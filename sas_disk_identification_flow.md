# SAS硬盘识别流程分析文档

## 1. 概述

本文档分析了Linux 2.6.27.19和3.10内核中使用PM8001驱动和libsas驱动的硬盘识别流程，重点关注了2.6版本中存在的问题以及3.10版本中的修复方案。

## 2. 2.6.27.19内核硬盘识别流程

### 2.1 PM8001驱动初始化流程

**注意：** 2.6.27.19内核中不存在PM8001驱动，只有libsas驱动。

### 2.2 libsas驱动硬盘识别流程

#### 2.2.1 函数调用链

```
scsi_scan_host
  └─> sas_scan_host
      └─> sas_discover_domain
          └─> sas_get_port_device
              └─> sas_discover_sata (对于SATA设备)
                  └─> sas_discover_sata_dev
                      └─> sas_issue_ata_cmd
                          └─> sas_execute_task  <-- 问题所在
```

#### 2.2.2 关键函数分析

**sas_execute_task函数**（/workspace/linux-2.6.27.19/drivers/scsi/libsas/sas_ata.c）：

```c
static int sas_execute_task(struct sas_task *task, void *buffer, int size,
                            enum dma_data_direction dma_dir)
{
    int res = 0;
    struct scatterlist *scatter = NULL;
    struct task_status_struct *ts = &task->task_status;
    int num_scatter = 0;
    int retries = 0;
    struct sas_internal *i =
        to_sas_internal(task->dev->port->ha->core.shost->transportt);

    // ... 初始化代码 ...

    for (retries = 0; retries < 5; retries++) {
        // ... 执行任务代码 ...

        if (task->task_status.stat == SAM_CHECK_COND) {
            struct scsi_sense_hdr shdr;

            if (!scsi_normalize_sense(ts->buf, ts->buf_valid_size,
                                      &shdr)) {
                SAS_DPRINTK("couldn't normalize sense\n");
                continue;
            }
            if ((shdr.sense_key == 6 && shdr.asc == 0x29) ||
                (shdr.sense_key == 2 && shdr.asc == 4 &&
                 shdr.ascq == 1)) {
                SAS_DPRINTK("device %016llx LUN: %016llx "
                            "powering up or not ready yet, "
                            "sleeping...\n",
                            SAS_ADDR(task->dev->sas_addr),
                            SAS_ADDR(task->ssp_task.LUN));

                schedule_timeout_interruptible(5*HZ);
            } else if (shdr.sense_key == 1) {
                res = 0;
                break;
            } else if (shdr.sense_key == 5) {
                break;
            } else {
                SAS_DPRINTK("dev %016llx LUN: %016llx "
                            "sense key:0x%x ASC:0x%x ASCQ:0x%x"
                            "\n",
                            SAS_ADDR(task->dev->sas_addr),
                            SAS_ADDR(task->ssp_task.LUN),
                            shdr.sense_key,
                            shdr.asc, shdr.ascq);
            }
        }
        // ... 其他错误处理 ...
    }
    // ... 清理代码 ...
    return res;
}
```

#### 2.2.3 问题标注

**问题：** 在`sas_execute_task`函数中，当处理sense key 0x02, 0x04, 0x01（磁盘正在旋转）时，代码会无限循环等待，没有重试次数限制。

**具体分析：**
- 当磁盘返回sense key 0x02, 0x04, 0x01时，表示磁盘正在旋转，需要等待
- 代码会调用`schedule_timeout_interruptible(5*HZ)`等待5秒
- 但是，这个等待逻辑在`for (retries = 0; retries < 5; retries++)`循环内部
- 当处理sense key 0x02, 0x04, 0x01时，代码没有`continue`或`break`语句，而是直接继续下一次循环
- 这导致即使设置了5次重试，当磁盘一直处于旋转状态时，会无限循环等待

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

**修复：** 3.10内核完全重构了硬盘识别流程，移除了有问题的`sas_execute_task`函数。

**具体改进：**
1. 采用事件驱动机制，使用`sas_discover_event`来处理各种发现事件
2. 移除了无限循环等待的逻辑
3. 改用更合理的错误处理方式，通过`ata_sas_async_probe`和标准的错误处理流程
4. 增加了设备状态管理和错误恢复机制

## 4. 版本对比

| 特性 | 2.6.27.19内核 | 3.10内核 |
|------|--------------|----------|
| PM8001驱动 | 不存在 | 存在 |
| 硬盘识别机制 | 同步执行，使用sas_execute_task | 事件驱动，使用sas_discover_event |
| 错误处理 | 无限循环等待，无重试限制 | 标准错误处理流程，有超时机制 |
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

Linux 2.6.27.19内核中存在SAS硬盘识别时的无限循环问题，主要原因是在`sas_execute_task`函数中处理磁盘旋转状态时没有重试次数限制。3.10内核通过重构整个硬盘识别流程，改用事件驱动机制和标准错误处理流程，彻底解决了这个问题。

建议用户升级到3.10或更高版本的内核，以避免这个问题的发生。对于无法升级内核的用户，可以采用临时修复方案，在`sas_execute_task`函数中添加重试次数限制。