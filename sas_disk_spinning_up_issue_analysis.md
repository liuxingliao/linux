# SAS硬盘启动问题分析报告

## 问题描述

在Linux 2.6.27.19内核中，当系统启动时遇到SAS硬盘有问题的情况，会出现以下错误信息：

```
Mar 16 16:07:38 SMH kernel: sd 8:0:5:0: [sdb] CDB: Read(10): 28 00 00 00 00 00 00 00 08 00
Mar 16 16:07:38 SMH kernel: sas (disk_io_done_eh_skey/752): device(0x5000c500f5cab04d),sense_key[0x02,0x04,0x01]
Mar 16 16:07:38 SMH kernel: sas (disk_io_done_eh_skey/753): scmd(ffff8800639720c0)
Mar 16 16:07:38 SMH kernel: sas (disk_io_done_eh_skey/774): h.skey[0x02,0x04,0x01],handler(ffffffffa0343ebe)
```

系统会卡在"Spinning up disk"状态，不断重复上述错误信息，导致无法继续启动。而在Linux 3.10和5.10内核中，这个问题不存在。

## 根本原因分析

通过对比不同版本内核的代码，发现问题出在`drivers/scsi/libsas/sas_ata.c`文件中：

### Linux 2.6.27.19内核

在2.6.27.19内核中，`sas_ata.c`文件包含以下代码：

```c
if ((shdr.sense_key == 6 && shdr.asc == 0x29) ||
    (shdr.sense_key == 2 && shdr.asc == 4 &&
     shdr.ascq == 1)) {
    SAS_DPRINTK("device %016llx LUN: %016llx "
                "powering up or not ready yet, "
                "sleeping...\n",
                SAS_ADDR(task->dev->sas_addr),
                SAS_ADDR(task->ssp_task.LUN));

    schedule_timeout_interruptible(5*HZ);
}
```

这段代码的问题在于：
1. 当检测到sense_key为0x02（Not Ready）、asc为0x04（Logical Unit Not Ready）、ascq为0x01（正在启动或旋转中）时
2. 代码会打印"powering up or not ready yet, sleeping..."信息
3. 然后调用`schedule_timeout_interruptible(5*HZ)`睡眠5秒
4. 但在某些情况下，这个逻辑会在一个循环中重复执行，导致无限睡眠

### Linux 3.10内核

在3.10内核中，这个逻辑已经被完全移除。3.10内核不再有这样的循环睡眠逻辑，而是将错误处理完全交给了libata的错误处理机制。

## 错误代码分析

### Sense Key解释

- **sense_key[0x02]**：Not Ready - 设备暂时无法就绪
- **asc[0x04]**：Logical Unit Not Ready - 逻辑单元暂时无法就绪
- **ascq[0x01]**：In Process of Becoming Ready - 设备正在准备就绪的过程中（如硬盘正在旋转启动）

### 问题机制

1. 系统启动时，尝试读取SAS硬盘
2. 硬盘处于"Spinning up"状态，返回Not Ready错误
3. 2.6.27.19内核的代码检测到这个错误，睡眠5秒
4. 睡眠后再次尝试读取，硬盘可能仍然处于启动状态
5. 重复上述过程，导致无限循环

## 修复方案

### 方案1：升级内核

最直接的解决方案是升级到Linux 3.10或更高版本的内核，这些版本已经修复了这个问题。

### 方案2：修改2.6.27.19内核代码

如果无法升级内核，可以修改`drivers/scsi/libsas/sas_ata.c`文件，移除或修改无限循环的逻辑：

1. **移除循环睡眠逻辑**：删除或注释掉导致无限睡眠的代码
2. **添加超时机制**：在循环中添加最大尝试次数，超过后放弃
3. **改进错误处理**：将错误处理交给更高级的错误处理机制

### 具体修改建议

在`drivers/scsi/libsas/sas_ata.c`文件中，找到类似以下的代码：

```c
while (1) {
    // 发送任务
    res = i->dft->lldd_execute_task(task, 0, GFP_KERNEL);
    if (res == 0) {
        sas_wait_for_task(task);
        if (task->task_status.resp == SAS_TASK_COMPLETE &&
            task->task_status.stat == SAM_GOOD)
            break;
        else if (task->task_status.stat == SAS_QUEUE_FULL) {
            SAS_DPRINTK("task: q busy, sleeping...\n");
            schedule_timeout_interruptible(HZ);
        } else if (task->task_status.stat == SAM_CHECK_COND) {
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
        } else if (task->task_status.resp != SAS_TASK_COMPLETE ||
                   task->task_status.stat != SAM_GOOD) {
            SAS_DPRINTK("task finished with resp:0x%x, "
                        "stat:0x%x\n",
                        task->task_status.resp,
                        task->task_status.stat);
        }
    } else {
        SAS_DPRINTK("lldd_execute_task returned %d\n", res);
    }
}
```

修改为：

```c
int retry_count = 0;
const int max_retries = 10; // 最大尝试次数

while (retry_count < max_retries) {
    retry_count++;
    // 发送任务
    res = i->dft->lldd_execute_task(task, 0, GFP_KERNEL);
    if (res == 0) {
        sas_wait_for_task(task);
        if (task->task_status.resp == SAS_TASK_COMPLETE &&
            task->task_status.stat == SAM_GOOD)
            break;
        else if (task->task_status.stat == SAS_QUEUE_FULL) {
            SAS_DPRINTK("task: q busy, sleeping...\n");
            schedule_timeout_interruptible(HZ);
        } else if (task->task_status.stat == SAM_CHECK_COND) {
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
                            "sleeping... (retry %d/%d)\n",
                            SAS_ADDR(task->dev->sas_addr),
                            SAS_ADDR(task->ssp_task.LUN),
                            retry_count, max_retries);

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
                break; // 其他错误直接 break
            }
        } else if (task->task_status.resp != SAS_TASK_COMPLETE ||
                   task->task_status.stat != SAM_GOOD) {
            SAS_DPRINTK("task finished with resp:0x%x, "
                        "stat:0x%x\n",
                        task->task_status.resp,
                        task->task_status.stat);
            break; // 其他错误直接 break
        }
    } else {
        SAS_DPRINTK("lldd_execute_task returned %d\n", res);
        break; // 执行任务失败直接 break
    }
}

if (retry_count >= max_retries) {
    SAS_DPRINTK("Max retries reached for device %016llx, giving up\n",
                SAS_ADDR(task->dev->sas_addr));
}
```

## 技术分析

### 为什么3.10和5.10内核不存在这个问题？

1. **架构改进**：3.10内核对SAS子系统进行了架构改进，将错误处理逻辑从libsas转移到了libata
2. **错误处理机制**：采用了更完善的错误处理机制，不再依赖简单的循环睡眠
3. **超时机制**：实现了更合理的超时机制，避免无限等待
4. **异步处理**：采用了更多的异步处理方式，提高了系统的响应性

### Sense Key 0x02,0x04,0x01 的含义

- **Sense Key 0x02**：Not Ready - 设备暂时无法处理命令
- **ASC 0x04**：Logical Unit Not Ready - 逻辑单元暂时无法就绪
- **ASCQ 0x01**：In Process of Becoming Ready - 设备正在准备过程中

这是硬盘启动时的正常状态，表示硬盘正在旋转启动，暂时无法接受命令。

## 总结

1. **问题根源**：Linux 2.6.27.19内核中SAS子系统的错误处理逻辑存在缺陷，当硬盘处于"Spinning up"状态时会无限循环睡眠

2. **修复方案**：
   - 升级到3.10或更高版本内核
   - 修改2.6.27.19内核代码，添加最大尝试次数限制

3. **技术教训**：
   - 错误处理逻辑应避免无限循环
   - 对于设备启动等临时性状态，应设置合理的超时机制
   - 应采用层次化的错误处理架构，将具体设备的错误处理交给专门的子系统

4. **影响范围**：
   - 仅影响Linux 2.6.27.19及更早版本的内核
   - 主要影响使用SAS硬盘的系统启动过程

通过理解这个问题的根本原因和修复方案，系统管理员和开发者可以更好地处理SAS硬盘的启动问题，确保系统能够正常启动和运行。