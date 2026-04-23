# SATA EVENT 0x23 处理流程分析

## 事件定义

**SATA EVENT 0x23** 对应宏定义：
```c
#define IO_XFER_ERROR_ABORTED_NCQ_MODE 0x23
```

## 处理流程

### 1. 事件触发

当PM80xx控制器检测到NCQ（Native Command Queuing）模式下的传输错误时，会触发 `IO_XFER_ERROR_ABORTED_NCQ_MODE` 事件（0x23）。

### 2. 事件处理流程

#### 第一步：mpi_sata_event 处理

```c
static void mpi_sata_event(struct pm8001_hba_info *pm8001_ha, void *piomb)
{
    // ...
    u32 event = le32_to_cpu(psataPayload->event);
    // ...
    
    /* Check if this is NCQ error */
    if (event == IO_XFER_ERROR_ABORTED_NCQ_MODE) {
        /* find device using device id */
        pm8001_dev = pm8001_find_dev(pm8001_ha, dev_id);
        /* send read log extension */
        if (pm8001_dev)
            pm8001_send_read_log(pm8001_ha, pm8001_dev);
        return;
    }
    // ...
}
```

**关键步骤**：
- 检测到 `IO_XFER_ERROR_ABORTED_NCQ_MODE` 事件
- 通过设备ID查找对应的设备
- 调用 `pm8001_send_read_log` 发送读取日志命令
- 直接返回，不继续处理其他逻辑

#### 第二步：pm8001_send_read_log 发送读取日志命令

```c
static void pm8001_send_read_log(struct pm8001_hba_info *pm8001_ha, struct pm8001_device *pm8001_ha_dev)
{
    // 分配任务
    task = sas_alloc_slow_task(GFP_ATOMIC);
    // ...
    // 构建FIS（Frame Information Structure）
    // ...
    // 发送SATA主机操作开始命令
    opc = OPC_INB_SATA_HOST_OPSTART;
    // ...
    // 设置NCQ_READ_LOG_FLAG标志
    pm8001_ha_dev->id |= NCQ_READ_LOG_FLAG;
    // ...
}
```

**关键步骤**：
- 分配一个慢任务
- 构建读取日志的FIS
- 发送SATA主机操作开始命令
- 在设备ID中设置 `NCQ_READ_LOG_FLAG` 标志

#### 第三步：mpi_sata_completion 处理读取日志响应

```c
static void mpi_sata_completion(struct pm8001_hba_info *pm8001_ha, void *piomb)
{
    // ...
    switch (status) {
    case IO_SUCCESS:
        // ...
        /* check if response is for SEND READ LOG */
        if (pm8001_dev && (pm8001_dev->id & NCQ_READ_LOG_FLAG)) {
            /* set new bit for abort_all */
            pm8001_dev->id |= NCQ_ABORT_ALL_FLAG;
            /* clear bit for read log */
            pm8001_dev->id = pm8001_dev->id & 0x7FFFFFFF;
            pm8001_send_abort_all(pm8001_ha, pm8001_dev);
            /* Free the tag */
            pm8001_tag_free(pm8001_ha, tag);
            sas_free_task(t);
            return;
        }
        // ...
    }
    // ...
}
```

**关键步骤**：
- 当读取日志命令成功完成（`IO_SUCCESS`）
- 检查是否是读取日志的响应（通过 `NCQ_READ_LOG_FLAG` 标志）
- 设置 `NCQ_ABORT_ALL_FLAG` 标志
- 清除 `NCQ_READ_LOG_FLAG` 标志
- 调用 `pm8001_send_abort_all` 中止所有任务
- 释放标签和任务

#### 第四步：pm8001_send_abort_all 中止所有任务

```c
static void pm8001_send_abort_all(struct pm8001_hba_info *pm8001_ha, struct pm8001_device *pm8001_ha_dev)
{
    // 分配任务
    task = sas_alloc_slow_task(GFP_ATOMIC);
    // ...
    // 构建任务中止请求
    // ...
    // 发送SATA中止命令
    opc = OPC_INB_SATA_ABORT;
    // ...
}
```

**关键步骤**：
- 分配一个慢任务
- 构建任务中止请求
- 发送SATA中止命令（`OPC_INB_SATA_ABORT`）

## 为什么要中止所有任务？

当发生 `IO_XFER_ERROR_ABORTED_NCQ_MODE` 事件时，说明NCQ模式下的传输出现了严重错误。这种情况下：

1. **首先读取日志**：通过 `pm8001_send_read_log` 读取设备的日志信息，获取错误的详细信息
2. **然后中止所有任务**：读取日志后，通过 `pm8001_send_abort_all` 中止所有正在进行的任务，以防止进一步的错误

## 代码流程示意图

```
┌─────────────────────┐
│ SATA EVENT 0x23     │
│ (NCQ模式错误)        │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ mpi_sata_event      │
│ - 检测到0x23事件    │
│ - 查找对应设备      │
│ - 调用pm8001_send_read_log │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ pm8001_send_read_log│
│ - 分配任务          │
│ - 构建读取日志FIS   │
│ - 发送SATA命令      │
│ - 设置NCQ_READ_LOG_FLAG │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 设备执行读取日志操作 │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ mpi_sata_completion │
│ - 处理读取日志响应  │
│ - 检测NCQ_READ_LOG_FLAG │
│ - 设置NCQ_ABORT_ALL_FLAG │
│ - 调用pm8001_send_abort_all │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ pm8001_send_abort_all│
│ - 分配任务          │
│ - 构建中止请求      │
│ - 发送SATA中止命令  │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 设备中止所有任务    │
└─────────────────────┘
```

## 日志分析

根据提供的日志：

```
[2026-04-10 20:32:35]<6>[  118.920259][    C8] pm80xx0:: mpi_sata_event  2840:SATA EVENT 0x23
[2026-04-10 20:32:35]<6>[  118.920264][    C8] pm80xx0:: pm80xx_send_read_log  1861:Executing read log end1
[2026-04-10 20:32:35]<6>[  118.920563][    C8] pm80xx0:: mpi_sata_event  2840:SATA EVENT 0x26
[2026-04-10 20:32:35]<6>[  118.920565][    C8] pm80xx0:: pm80xx_send_abort_all  1785:Executing abort task end2
[2026-04-10 20:32:35]<6>[  118.920619][    C8] pm80xx0:: pm8001_mpi_task_abort_resp  3737:IO_SUCCESS
[2026-04-10 20:32:35]<6>[  118.920623][   C12] pm80xx0:: mpi_sata_completion  2444:status:0x1, tag:0x2, task::0xffff8881d70b8b00
[2026-04-10 20:32:35]<6>[  118.920625][   C12] pm80xx0:: mpi_sata_completion  2484:SAS Address of IO Failure Drive:604eecd1b27dc312
[2026-04-10 20:32:35]<4>[  118.920627][   C12] sas: sas_ata_task_done: SAS error 0x8d
```

**日志流程分析**：
1. 首先触发了 `SATA EVENT 0x23`（NCQ模式错误）
2. 执行 `pm80xx_send_read_log` 读取日志
3. 然后触发了 `SATA EVENT 0x26`（可能是读取日志的事件）
4. 执行 `pm80xx_send_abort_all` 中止所有任务
5. 中止任务成功（`IO_SUCCESS`）
6. 随后多个 `mpi_sata_completion` 调用，报告任务完成状态为0x1（错误）
7. 最终 `sas_ata_task_done` 报告 SAS 错误 0x8d

## 结论

当SATA设备在NCQ模式下出现传输错误时，PM80xx驱动的处理流程是：

1. **读取错误日志**：首先通过 `pm8001_send_read_log` 读取设备的错误日志，获取详细的错误信息
2. **中止所有任务**：读取日志后，通过 `pm8001_send_abort_all` 中止所有正在进行的任务，以防止错误扩散
3. **报告错误**：最后通过 `sas_ata_task_done` 向上层报告SAS错误

这种处理方式确保了在NCQ模式出现错误时，能够先获取错误信息，然后清理所有相关任务，最后将错误报告给上层，从而保证系统的稳定性和错误的可追溯性。