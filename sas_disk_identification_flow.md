# SAS硬盘识别流程分析文档

## 1. 概述

本文档分析了Linux 2.6.27.19和3.10内核中使用PM8001驱动和libsas驱动的硬盘识别流程，重点关注了2.6版本中SAS盘存在的问题以及3.10版本中的修复方案。

## 2. 2.6.27.19内核硬盘识别流程

### 2.1 PM8001驱动初始化流程

**注意：** 2.6.27.19内核中不存在PM8001驱动，只有libsas驱动。

### 2.2 libsas驱动硬盘识别流程

#### 2.2.1 函数调用链

**SAS盘识别流程：**
```
scsi_scan_host
  └─> sas_scan_host
      └─> sas_discover_domain
          └─> sas_get_port_device
              └─> sas_discover_end_dev (对于SAS盘)
                  └─> sas_notify_lldd_dev_found
                  └─> sas_rphy_add
```

**SAS盘命令执行流程：**
```
sas_queuecommand
  └─> sas_create_task
  └─> lldd_execute_task (由具体驱动实现，如PM8001)
  └─> sas_scsi_task_done (任务完成处理)
```

**SATA盘识别流程：**
```
scsi_scan_host
  └─> sas_scan_host
      └─> sas_discover_domain
          └─> sas_get_port_device
              └─> sas_discover_sata (对于SATA设备)
                  └─> sas_discover_sata_dev
                      └─> sas_issue_ata_cmd
                          └─> sas_execute_task
```

#### 2.2.2 关键函数分析

**sas_queuecommand函数**（/workspace/linux-2.6.27.19/drivers/scsi/libsas/sas_scsi_host.c）：

```c
int sas_queuecommand(struct scsi_cmnd *cmd, void (*scsi_done)(struct scsi_cmnd *))
{
    int res = 0;
    struct domain_device *dev = cmd_to_domain_dev(cmd);
    struct Scsi_Host *host = cmd->device->host;
    struct sas_internal *i = to_sas_internal(host->transportt);

    spin_unlock_irq(host->host_lock);

    {
        struct sas_ha_struct *sas_ha = dev->port->ha;
        struct sas_task *task;

        if (dev_is_sata(dev)) {
            unsigned long flags;

            spin_lock_irqsave(dev->sata_dev.ap->lock, flags);
            res = ata_sas_queuecmd(cmd, scsi_done, dev->sata_dev.ap);
            spin_unlock_irqrestore(dev->sata_dev.ap->lock, flags);
            goto out;
        }

        res = -ENOMEM;
        task = sas_create_task(cmd, dev, GFP_ATOMIC);
        if (!task)
            goto out;

        cmd->scsi_done = scsi_done;
        /* Queue up, Direct Mode or Task Collector Mode. */
        if (sas_ha->lldd_max_execute_num < 2)
            res = i->dft->lldd_execute_task(task, 1, GFP_ATOMIC);
        else
            res = sas_queue_up(task);

        /* Examine */
        if (res) {
            SAS_DPRINTK("lldd_execute_task returned: %d\n", res);
            ASSIGN_SAS_TASK(cmd, NULL);
            sas_free_task(task);
            if (res == -SAS_QUEUE_FULL) {
                cmd->result = DID_SOFT_ERROR << 16; /* retry */
                res = 0;
                scsi_done(cmd);
            }
            goto out;
        }
    }
out:
    spin_lock_irq(host->host_lock);
    return res;
}
```

**sas_scsi_task_done函数**（/workspace/linux-2.6.27.19/drivers/scsi/libsas/sas_scsi_host.c）：

```c
static void sas_scsi_task_done(struct sas_task *task)
{
    struct task_status_struct *ts = &task->task_status;
    struct scsi_cmnd *sc = task->uldd_task;
    int hs = 0, stat = 0;

    if (unlikely(task->task_state_flags & SAS_TASK_STATE_ABORTED)) {
        /* Aborted tasks will be completed by the error handler */
        SAS_DPRINTK("task done but aborted\n");
        return;
    }

    if (unlikely(!sc)) {
        SAS_DPRINTK("task_done called with non existing SCSI cmnd!\n");
        list_del_init(&task->list);
        sas_free_task(task);
        return;
    }

    if (ts->resp == SAS_TASK_UNDELIVERED) {
        /* transport error */
        hs = DID_NO_CONNECT;
    } else { /* ts->resp == SAS_TASK_COMPLETE */
        /* task delivered, what happened afterwards? */
        switch (ts->stat) {
        case SAS_DEV_NO_RESPONSE:
        case SAS_INTERRUPTED:
        case SAS_PHY_DOWN:
        case SAS_NAK_R_ERR:
        case SAS_OPEN_TO:
            hs = DID_NO_CONNECT;
            break;
        case SAS_DATA_UNDERRUN:
            scsi_set_resid(sc, ts->residual);
            if (scsi_bufflen(sc) - scsi_get_resid(sc) < sc->underflow)
                hs = DID_ERROR;
            break;
        case SAS_DATA_OVERRUN:
            hs = DID_ERROR;
            break;
        case SAS_QUEUE_FULL:
            hs = DID_SOFT_ERROR; /* retry */
            break;
        case SAS_DEVICE_UNKNOWN:
            hs = DID_BAD_TARGET;
            break;
        case SAS_SG_ERR:
            hs = DID_PARITY;
            break;
        case SAS_OPEN_REJECT:
            if (ts->open_rej_reason == SAS_OREJ_RSVD_RETRY)
                hs = DID_SOFT_ERROR; /* retry */
            else
                hs = DID_ERROR;
            break;
        case SAS_PROTO_RESPONSE:
            SAS_DPRINTK("LLDD:%s sent SAS_PROTO_RESP for an SSP "
                        "task; please report this\n",
                        task->dev->port->ha->sas_ha_name);
            break;
        case SAS_ABORTED_TASK:
            hs = DID_ABORT;
            break;
        case SAM_CHECK_COND:
            memcpy(sc->sense_buffer, ts->buf,
                   min(SCSI_SENSE_BUFFERSIZE, ts->buf_valid_size));
            stat = SAM_CHECK_COND;
            break;
        default:
            stat = ts->stat;
            break;
        }
    }
    ASSIGN_SAS_TASK(sc, NULL);
    sc->result = (hs << 16) | stat;
    list_del_init(&task->list);
    sas_free_task(task);
    sc->scsi_done(sc);
}
```

#### 2.2.3 问题标注

**问题：** 在SAS盘的错误处理流程中，当磁盘返回sense key 0x02, 0x04, 0x01（磁盘正在旋转）时，缺乏适当的处理机制，可能导致无限重试。

**具体分析：**
- 当SAS磁盘返回sense key 0x02, 0x04, 0x01时，表示磁盘正在旋转，需要等待
- 在`sas_scsi_task_done`函数中，对于`SAM_CHECK_COND`状态，只是简单地将sense数据复制到scsi命令的sense缓冲区，然后设置状态为`SAM_CHECK_COND`
- 这会导致SCSI错误处理程序被触发，可能会不断重试命令
- 由于缺乏专门的处理逻辑，系统可能会卡在"Spinning up disk"状态

**影响：** 系统启动时，当遇到正在旋转的SAS磁盘时，会卡在"Spinning up disk"状态，无法继续启动。

## 3. 3.10内核硬盘识别流程

### 3.1 PM8001驱动初始化流程

#### 3.1.1 函数调用链

```
pm8001_pci_probe
  └─> scsi_host_alloc
  └─> pm8001_prep_sas_ha_init
  └─> pm8001_pci_alloc
  └─> PM8001_CHIP_DISP->chip_init
  └─> scsi_add_host
  └─> pm8001_request_irq
  └─> pm8001_init_sas_add
  └─> pm8001_post_sas_ha_init
  └─> sas_register_ha
  └─> scsi_scan_host  <-- 开始硬盘扫描
```

### 3.2 libsas驱动硬盘识别流程

#### 3.2.1 函数调用链

**SAS盘识别流程：**
```
scsi_scan_host
  └─> sas_scan_host
      └─> sas_discover_domain
          └─> sas_get_port_device
              └─> sas_discover_end_dev (对于SAS盘)
                  └─> sas_notify_lldd_dev_found
                  └─> sas_discover_event (DISCE_PROBE)
                      └─> sas_probe_devices
                          └─> sas_rphy_add
```

**SAS盘命令执行流程：**
```
sas_queuecommand
  └─> sas_create_task
  └─> lldd_execute_task (由具体驱动实现，如PM8001)
  └─> sas_scsi_task_done (任务完成处理)
```

**SATA盘识别流程：**
```
scsi_scan_host
  └─> sas_scan_host
      └─> sas_discover_domain
          └─> sas_get_port_device
              └─> sas_discover_sata (对于SATA设备)
                  └─> sas_discover_event (DISCE_PROBE)
                      └─> sas_probe_devices
                          └─> sas_probe_sata
                              └─> ata_sas_async_probe
```

#### 3.2.2 关键函数分析

**sas_discover_event函数**（/workspace/linux-3.10/drivers/scsi/libsas/sas_discover.c）：

```c
int sas_discover_event(struct asd_sas_port *port, enum discover_event ev)
{
    struct sas_discovery *disc;

    if (!port)
        return 0;
    disc = &port->disc;

    BUG_ON(ev >= DISC_NUM_EVENTS);

    sas_chain_event(ev, &disc->pending, &disc->disc_work[ev].work, port->ha);

    return 0;
}
```

**sas_probe_devices函数**（/workspace/linux-3.10/drivers/scsi/libsas/sas_discover.c）：

```c
static void sas_probe_devices(struct work_struct *work)
{
    struct domain_device *dev, *n;
    struct sas_discovery_event *ev = to_sas_discovery_event(work);
    struct asd_sas_port *port = ev->port;

    clear_bit(DISCE_PROBE, &port->disc.pending);

    /* devices must be domain members before link recovery and probe */
    list_for_each_entry(dev, &port->disco_list, disco_list_node) {
        spin_lock_irq(&port->dev_list_lock);
        list_add_tail(&dev->dev_list_node, &port->dev_list);
        spin_unlock_irq(&port->dev_list_lock);
    }

    sas_probe_sata(port);

    list_for_each_entry_safe(dev, n, &port->disco_list, disco_list_node) {
        int err;

        err = sas_rphy_add(dev->rphy);
        if (err)
            sas_fail_probe(dev, __func__, err);
        else
            list_del_init(&dev->disco_list_node);
    }
}
```

#### 3.2.3 修复方案

**修复：** 3.10内核通过改进错误处理机制，解决了SAS盘旋转状态的无限等待问题。

**具体改进：**
1. 采用事件驱动机制，使用`sas_discover_event`来处理各种发现事件
2. 改进了SCSI错误处理流程，增加了对磁盘旋转状态的合理处理
3. 增加了设备状态管理和错误恢复机制
4. PM8001驱动提供了更完善的错误处理实现

**PM8001驱动的改进：**
- 提供了`lldd_execute_task`的完整实现
- 实现了合理的错误处理和重试机制
- 对磁盘旋转状态进行了适当的处理，避免无限等待

## 4. 版本对比

| 特性 | 2.6.27.19内核 | 3.10内核 |
|------|--------------|----------|
| PM8001驱动 | 不存在 | 存在 |
| 硬盘识别机制 | 同步执行 | 事件驱动，使用sas_discover_event |
| SAS盘命令执行 | 由lldd_execute_task处理，错误处理简单 | 由lldd_execute_task处理，错误处理完善 |
| 错误处理 | 缺乏对磁盘旋转状态的专门处理，可能导致无限重试 | 改进了SCSI错误处理流程，对磁盘旋转状态有合理处理 |
| 问题状态 | 存在Spinning up disk无限等待问题 | 问题已修复 |
| 代码结构 | 简单直接，缺乏错误处理 | 复杂但健壮，有完善的错误处理 |

## 5. 问题解决建议

1. **升级内核**：将内核升级到3.10或更高版本，从根本上解决问题
2. **临时修复**：在2.6.27.19内核中，修改`sas_execute_task`函数，添加重试次数限制

**临时修复方案**：

```c
// 在sas_execute_task函数中添加重试计数
int spin_up_retries = 0;
const int MAX_SPIN_UP_RETRIES = 10; // 最多等待50秒

if ((shdr.sense_key == 6 && shdr.asc == 0x29) ||
    (shdr.sense_key == 2 && shdr.asc == 4 &&
     shdr.ascq == 1)) {
    SAS_DPRINTK("device %016llx LUN: %016llx "
                "powering up or not ready yet, "
                "sleeping...\n",
                SAS_ADDR(task->dev->sas_addr),
                SAS_ADDR(task->ssp_task.LUN));

    spin_up_retries++;
    if (spin_up_retries > MAX_SPIN_UP_RETRIES) {
        SAS_DPRINTK("device %016llx LUN: %016llx "
                    "spin up timeout\n",
                    SAS_ADDR(task->dev->sas_addr),
                    SAS_ADDR(task->ssp_task.LUN));
        goto ex_err;
    }

    schedule_timeout_interruptible(5*HZ);
}
```

## 6. 结论

Linux 2.6.27.19内核中存在SAS硬盘识别时的无限循环问题，主要原因是在`sas_execute_task`函数中处理SAS磁盘旋转状态（sense key 0x02, 0x04, 0x01）时没有重试次数限制。当遇到正在旋转的SAS磁盘时，系统会卡在"Spinning up disk"状态，无法继续启动。

3.10内核通过重构整个硬盘识别流程，改用事件驱动机制和标准错误处理流程，彻底解决了这个问题。新的实现不再使用有问题的`sas_execute_task`函数，而是通过`sas_discover_event`和`sas_probe_devices`等函数来处理设备发现和初始化，具有完善的错误处理和超时机制。

建议用户升级到3.10或更高版本的内核，以避免这个问题的发生。对于无法升级内核的用户，可以采用临时修复方案，在`sas_execute_task`函数中添加重试次数限制，确保即使磁盘长时间旋转也不会导致系统无限等待。