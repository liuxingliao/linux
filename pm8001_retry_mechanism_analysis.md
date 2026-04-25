# PM8001驱动IO错误重试机制分析

## 1. 概述

本文档分析Linux内核5.10版本中PM8001 SAS/SATA HBA驱动在IO错误发生时的重试机制，涵盖了SCSI层、libsas层和PM8001驱动层三个层面的处理逻辑。

## 1.1 SATA设备NCQ错误处理流程

当SATA设备发生NCQ错误时，PM8001驱动会执行以下流程：

1. **硬件事件触发**：固件上报 `SATA EVENT 0x23` (IO_XFER_ERROR_ABORTED_NCQ_MODE)
2. **发送Read Log命令**：调用 `pm80xx_send_read_log` 发送ATA READ LOG EXT命令读取log page 0x10
3. **Abort所有命令**：Read Log完成后，调用 `pm80xx_send_abort_all` abort该设备上的所有待处理命令
4. **错误恢复**：通过libata/SCSI错误处理机制进行恢复

---

## 2. SCSI中层重试机制

### 2.1 核心文件
- [`/workspace/drivers/scsi/scsi_error.c`](file:///workspace/drivers/scsi/scsi_error.c)

### 2.2 重试判断逻辑

#### 2.2.1 `scsi_cmd_retry_allowed`函数
```c
static bool scsi_cmd_retry_allowed(struct scsi_cmnd *cmd)
{
    if (cmd->allowed == SCSI_CMD_RETRIES_NO_LIMIT)
        return true;
    return ++cmd->retries <= cmd->allowed;
}
```
- 检查命令是否允许重试
- 支持无限制重试或指定次数重试

#### 2.2.2 错误处理流程
1. 命令超时或错误 → `scsi_times_out`
2. 尝试中止命令 → `scsi_try_to_abort_cmd`
3. 若中止成功且允许重试 → 重新入队 `scsi_queue_insert(cmd, SCSI_MLQUEUE_EH_RETRY)`
4. 否则完成命令 → `scsi_finish_command`

### 2.3 错误处理状态机
SCSI中层实现了完整的错误恢复流程：
- 主机进入错误恢复状态
- 处理失败的命令队列
- 尝试各种恢复操作（中止、重置等）
- 决定是否重试

---

## 3. libsas层重试机制

### 3.1 核心文件
- [`/workspace/drivers/scsi/libsas/sas_scsi_host.c`](file:///workspace/drivers/scsi/libsas/sas_scsi_host.c)

### 3.2 状态码与重试映射

| SAS状态码 | 主机字节码 | 说明 | 是否重试 |
|----------|----------|-----|---------|
| `SAS_QUEUE_FULL` | `DID_SOFT_ERROR` | 队列满 | 是 |
| `SAS_OPEN_REJECT` | `DID_SOFT_ERROR` | 打开被拒绝（原因=`SAS_OREJ_RSVD_RETRY`） | 是 |
| `SAS_PHY_DOWN` | `DID_NO_CONNECT` | 物理层断开 | 否 |
| `SAS_ABORTED_TASK` | `DID_ABORT` | 任务被中止 | 否 |

### 3.3 关键代码实现

#### 3.3.1 `sas_end_task` 函数
```c
static void sas_end_task(struct scsi_cmnd *sc, struct sas_task *task)
{
    struct task_status_struct *ts = &task->task_status;
    int hs = 0, stat = 0;

    if (ts->resp == SAS_TASK_UNDELIVERED) {
        hs = DID_NO_CONNECT;
    } else {
        switch (ts->stat) {
        case SAS_QUEUE_FULL:
            hs = DID_SOFT_ERROR; // 指示上层重试
            break;
        case SAS_OPEN_REJECT:
            if (ts->open_rej_reason == SAS_OREJ_RSVD_RETRY)
                hs = DID_SOFT_ERROR; // 指示上层重试
            else
                hs = DID_ERROR;
            break;
        // ... 其他状态处理
        }
    }
    sc->result = (hs << 16) | stat;
    // ...
}
```

#### 3.3.2 `sas_queuecommand` 函数
```c
int sas_queuecommand(struct Scsi_Host *host, struct scsi_cmnd *cmd)
{
    // ...
    if (res == -SAS_QUEUE_FULL)
        cmd->result = DID_SOFT_ERROR << 16; // 重试
    else
        cmd->result = DID_ERROR << 16;
out_done:
    cmd->scsi_done(cmd);
    return 0;
}
```

### 3.4 libsas错误恢复流程

#### 3.4.1 任务级恢复
1. 查找任务 → `sas_scsi_find_task`
2. 中止任务 → `lldd_abort_task`
3. 查询任务状态 → `lldd_query_task`

#### 3.4.2 LUN级恢复
1. 中止任务集 → `lldd_abort_task_set`
2. 清除任务集 → `lldd_clear_task_set`
3. LUN重置 → `lldd_lu_reset`

#### 3.4.3 I_T Nexus级恢复
1. I_T Nexus重置 → `lldd_I_T_nexus_reset`

---

## 4. PM8001驱动层重试机制

### 4.1 核心文件
- [`/workspace/drivers/scsi/pm8001/pm8001_sas.c`](file:///workspace/drivers/scsi/pm8001/pm8001_sas.c)

### 4.2 打开拒绝重试机制

#### 4.2.1 `pm8001_open_reject_retry`函数
```c
void pm8001_open_reject_retry(
    struct pm8001_hba_info *pm8001_ha,
    struct sas_task *task_to_close,
    struct pm8001_device *device_to_close)
{
    // ...
    for (i = 0; i < PM8001_MAX_CCB; i++) {
        // ...
        ts->resp = SAS_TASK_COMPLETE;
        /* 强制中层重试 */
        ts->stat = SAS_OPEN_REJECT;
        ts->open_rej_reason = SAS_OREJ_RSVD_RETRY;
        // ...
        task->task_done(task);
        // ...
    }
}
```

**功能说明**：
- 当发生特定错误（如打开被拒绝）时，该函数会强制将任务状态设置为可重试状态
- 通过设置 `SAS_OPEN_REJECT` 和 `SAS_OREJ_RSVD_RETRY` 来通知上层重试
- 可以针对特定任务、特定设备或整个HBA的所有任务进行重试

### 4.3 内部TMF任务重试

#### 4.3.1 `pm8001_exec_internal_tmf_task`函数
```c
static int pm8001_exec_internal_tmf_task(struct domain_device *dev,
    void *parameter, u32 para_len, struct pm8001_tmf_task *tmf)
{
    int res, retry;
    // ...
    for (retry = 0; retry < 3; retry++) {  // 最多重试3次
        // ... 执行TMF任务
        if (task->task_status.resp == SAS_TASK_COMPLETE &&
            task->task_status.stat == SAM_STAT_GOOD) {
            res = TMF_RESP_FUNC_COMPLETE;
            break;
        }
        // ...
    }
    // ...
}
```

### 4.4 SATA设备特殊处理

对于SATA设备，PM8001驱动提供了更全面的错误处理：

1. **设备状态设置**：将设备设置为恢复状态
2. **PHY硬重置**：发送PHY控制硬重置命令
3. **中止所有任务**：执行内部SATA中止所有命令
4. **恢复设备状态**：将设备恢复为操作状态

---

## 5. 完整调用流程

### 5.1 正常IO重试流程（队列满为例）

```
应用层
   ↓
块设备层
   ↓
SCSI中层 (scsi_dispatch_cmd)
   ↓
libsas层 (sas_queuecommand)
   ↓
PM8001驱动 (pm8001_queue_command)
   ↓
PM8001硬件 → 检测到队列满 (返回 -SAS_QUEUE_FULL)
   ↓
PM8001驱动 → 设置 cmd->result = DID_SOFT_ERROR << 16
   ↓
libsas层 → sas_scsi_task_done → sas_end_task
   ↓
SCSI中层 → scsi_done → 判断为可重试错误
   ↓
scsi_cmd_retry_allowed → 检查重试次数
   ↓
scsi_queue_insert → 重新入队，等待再次调度
```

### 5.2 打开拒绝重试流程

```
PM8001硬件 → 检测到打开被拒绝事件
   ↓
PM8001驱动中断处理
   ↓
pm8001_open_reject_retry(ha, NULL, NULL)  // 重试所有相关任务
   ↓
遍历所有CCB，设置状态为:
   - ts->resp = SAS_TASK_COMPLETE
   - ts->stat = SAS_OPEN_REJECT
   - ts->open_rej_reason = SAS_OREJ_RSVD_RETRY
   ↓
调用 task->task_done(task)
   ↓
libsas层 → sas_end_task → 识别为可重试错误
   ↓
SCSI中层 → 重新入队
```

### 5.3 错误恢复后的重试流程

```
IO错误发生
   ↓
SCSI EH唤醒 (scsi_eh_wakeup)
   ↓
libsas错误处理 (sas_scsi_recover_host)
   ↓
PM8001驱动TMF操作 (lldd_abort_task, lldd_lu_reset等)
   ↓
恢复成功
   ↓
SCSI中层 → 重新入队未完成的可重试命令
```

---

## 6. 总结

### 6.1 各层职责

| 层级 | 职责 |
|-----|------|
| **SCSI中层** | 提供统一的错误处理框架，管理重试次数，决定何时放弃重试 |
| **libsas层** | SAS协议特定的错误处理，状态码映射，SAS设备恢复操作 |
| **PM8001驱动** | 硬件特定的错误恢复，打开拒绝重试，内部TMF重试 |

### 6.2 重试触发条件

1. **队列满** (`SAS_QUEUE_FULL` → `DID_SOFT_ERROR`)
2. **打开被拒绝（可重试）** (`SAS_OREJ_RSVD_RETRY` → `DID_SOFT_ERROR`)
3. **临时错误**（通过错误恢复流程解决后）

### 6.3 关键特性

1. **分层设计**：各层职责清晰，便于维护和扩展
2. **渐进式恢复**：从任务级→LUN级→I_T Nexus级，逐步升级恢复措施
3. **智能重试**：根据错误类型决定是否重试，避免不必要的重试
4. **硬件辅助**：PM8001硬件提供特定的错误恢复支持

---

## 7. 介质错误处理机制

### 7.1 介质错误概述
介质错误（MEDIUM_ERROR）是指存储介质本身的物理问题，如坏道、不可恢复的读写错误等。这类错误通常通过SCSI的CHECK CONDITION状态返回，并在sense data中标识。

### 7.2 SCSI层介质错误处理

#### 7.2.1 `scsi_check_sense`函数中的处理
在 [`scsi_error.c`](file:///workspace/drivers/scsi/scsi_error.c#L618-L625) 中，介质错误的处理逻辑为：

```c
case MEDIUM_ERROR:
    if (sshdr.asc == 0x11 || /* UNRECOVERED READ ERR */
        sshdr.asc == 0x13 || /* AMNF DATA FIELD */
        sshdr.asc == 0x14) { /* RECORD NOT FOUND */
        set_host_byte(scmd, DID_MEDIUM_ERROR);
        return SUCCESS;
    }
    return NEEDS_RETRY;
```

**处理逻辑**：
- **不可恢复的介质错误**（ASC=0x11, 0x13, 0x14）：直接返回 `SUCCESS`，设置 `DID_MEDIUM_ERROR`，**不重试**
- **其他介质错误**：返回 `NEEDS_RETRY`，**允许重试**

#### 7.2.2 不可恢复介质错误类型
| ASC码 | 说明 | 是否重试 |
|-------|------|---------|
| 0x11 | UNRECOVERED READ ERROR | 否 |
| 0x13 | AMNF DATA FIELD | 否 |
| 0x14 | RECORD NOT FOUND | 否 |

### 7.3 libsas层介质错误传递
在 [`sas_scsi_host.c`](file:///workspace/drivers/scsi/libsas/sas_scsi_host.c#L85-L92) 中，当任务完成且状态为 `SAM_STAT_CHECK_CONDITION` 时，会将sense data复制到SCSI命令中：

```c
case SAM_STAT_CHECK_CONDITION:
    memcpy(sc->sense_buffer, ts->buf,
           min(SCSI_SENSE_BUFFERSIZE, ts->buf_valid_size));
    stat = SAM_STAT_CHECK_CONDITION;
    break;
```

libsas层本身不处理介质错误的具体逻辑，只是将sense data透传给SCSI中层。

### 7.4 块层对介质错误的处理
在 [`scsi_lib.c`](file:///workspace/drivers/scsi/scsi_lib.c#L642-L644) 中，`DID_MEDIUM_ERROR` 被映射为块层的 `BLK_STS_MEDIUM` 错误：

```c
case DID_MEDIUM_ERROR:
    set_host_byte(cmd, DID_OK);
    return BLK_STS_MEDIUM;
```

### 7.5 SCSI磁盘驱动处理
在 [`sd.c`](file:///workspace/drivers/scsi/sd.c#L2078-L2081) 中，当发生介质错误或硬件错误时，会计算已完成的字节数：

```c
case HARDWARE_ERROR:
case MEDIUM_ERROR:
    good_bytes = sd_completed_bytes(SCpnt);
    break;
```

### 7.6 介质错误重试机制总结

| 介质错误类型 | 是否重试 | 处理方式 |
|------------|---------|---------|
| 可恢复的介质错误（非0x11/0x13/0x14） | 是 | 返回 `NEEDS_RETRY`，SCSI中层会尝试重试 |
| 不可恢复的读错误（ASC=0x11） | 否 | 返回 `SUCCESS`，设置 `DID_MEDIUM_ERROR` |
| AMNF数据字段错误（ASC=0x13） | 否 | 返回 `SUCCESS`，设置 `DID_MEDIUM_ERROR` |
| 记录未找到（ASC=0x14） | 否 | 返回 `SUCCESS`，设置 `DID_MEDIUM_ERROR` |

**设计理由**：
- 不可恢复的介质错误重试没有意义，只会浪费时间
- 某些介质错误可能是临时的（如介质抖动），允许重试
- 通过sense data中的具体ASC/ASCQ区分可恢复和不可恢复的情况

---

## 8. Abort I/O处理机制

### 8.1 Abort I/O概述
Abort I/O是指由于各种原因（如超时、设备错误等）导致的任务被中止的情况。这类错误通常通过 `DID_ABORT` 主机字节码标识。

### 8.2 SCSI层Abort I/O处理

在 [`scsi_error.c`](file:///workspace/drivers/scsi/scsi_error.c#L1823-L1828) 中，Abort I/O的处理逻辑为：

```c
case DID_ABORT:
    if (scmd->eh_eflags & SCSI_EH_ABORT_SCHEDULED) {
        set_host_byte(scmd, DID_TIME_OUT);
        return SUCCESS;
    }
    fallthrough;
case DID_NO_CONNECT:
case DID_BAD_TARGET:
    return SUCCESS;
```

**处理逻辑**：
- **常规Abort**：直接返回 `SUCCESS`，**不重试**
- **错误处理期间的Abort**（`SCSI_EH_ABORT_SCHEDULED`）：将状态改为 `DID_TIME_OUT` 后返回 `SUCCESS`，**不重试**

### 8.3 libsas层Abort I/O处理

在 [`sas_scsi_host.c`](file:///workspace/drivers/scsi/libsas/sas_scsi_host.c#L82-L84) 中，当任务状态为 `SAS_ABORTED_TASK` 时：

```c
case SAS_ABORTED_TASK:
    hs = DID_ABORT;
    break;
```

libsas层将 `SAS_ABORTED_TASK` 状态映射为 `DID_ABORT` 主机字节码，传递给SCSI中层处理。

### 8.4 PM8001驱动层Abort处理

PM8001驱动提供了完整的abort机制：

1. **任务级Abort**：`pm8001_abort_task` 函数
2. **设备级Abort**：`pm8001_exec_internal_task_abort` 函数（支持abort单个任务或所有任务）
3. **错误恢复中的Abort**：在I_T Nexus重置等操作中使用

### 8.5 Abort I/O重试机制总结

| 场景 | 是否重试 | 处理方式 |
|------|---------|---------|
| 常规Abort | 否 | 返回 `SUCCESS`，设置 `DID_ABORT` |
| 错误处理期间的Abort | 否 | 改为 `DID_TIME_OUT`，返回 `SUCCESS` |
| SAS_ABORTED_TASK | 否 | 映射为 `DID_ABORT`，不重试 |

**设计理由**：
- Abort通常意味着任务已经被终止，重试没有意义
- 某些Abort可能是由于硬件错误或资源问题导致的，需要通过错误恢复流程解决
- 错误处理期间的Abort会被视为超时，由错误处理流程处理

### 8.6 Abort与错误恢复的关系

当发生Abort时，SCSI中层会：
1. 标记命令为完成状态
2. 不进行重试
3. 如果是在错误处理期间发生的Abort，会将其视为超时
4. 错误处理流程会尝试通过各种恢复操作（如设备重置、总线重置等）来恢复系统状态

对于PM8001驱动，当检测到需要abort任务时：
1. 发送abort命令到硬件
2. 等待abort完成
3. 清理相关资源
4. 通知上层任务已完成（状态为 `DID_ABORT`）

**注意**：Abort本身不会触发重试，但如果在错误恢复成功后，其他未完成的命令可能会被重新调度。

---

## 9. SATA盘IO Abort处理机制

### 9.1 SATA盘与SAS盘的区别

SAS盘使用 `SAS_PROTOCOL_SSP`（SCSI SAS Protocol），而SATA盘使用：
- `SAS_PROTOCOL_SATA`（直接SATA）
- `SAS_PROTOCOL_STP`（SATA Tunneling Protocol）

### 9.2 PM8001驱动中SATA任务准备

在 [`pm8001_sas.c`](file:///workspace/drivers/scsi/pm8001/pm8001_sas.c#L469-L470) 中，SATA任务使用 `pm8001_task_prep_ata` 准备：

```c
case SAS_PROTOCOL_SATA:
case SAS_PROTOCOL_STP:
    rc = pm8001_task_prep_ata(pm8001_ha, ccb);
    break;
```

### 9.3 SATA任务完成处理

SATA任务完成后，通过 [`sas_ata.c`](file:///workspace/drivers/scsi/libsas/sas_ata.c#L81-L156) 中的 `sas_ata_task_done` 函数处理：

```c
static void sas_ata_task_done(struct sas_task *task)
{
    struct ata_queued_cmd *qc = task->uldd_task;
    // ...
    if (stat->stat == SAS_PROTO_RESPONSE || stat->stat == SAM_STAT_GOOD ||
        ((stat->stat == SAM_STAT_CHECK_CONDITION &&
          dev->sata_dev.class == ATA_DEV_ATAPI))) {
        // 处理正常响应
    } else {
        ac = sas_to_ata_err(stat);
        // ...
    }
    // ...
    ata_qc_complete(qc);
}
```

### 9.4 SATA盘Abort状态映射

在 [`sas_ata.c`](file:///workspace/drivers/scsi/libsas/sas_ata.c#L68-L70) 中，`sas_to_ata_err` 函数将 `SAS_ABORTED_TASK` 映射为 `AC_ERR_DEV`：

```c
case SAM_STAT_CHECK_CONDITION:
case SAS_ABORTED_TASK:
    return AC_ERR_DEV;
```

### 9.5 SATA盘IO Abort的scmd->result状态

**关键区别**：对于SATA盘，由于使用libata框架处理，**不会直接设置 `scmd->result = DID_ABORT << 16`**。

SATA盘的完整处理流程：

1. **状态映射**：在 [`sas_ata.c`](file:///workspace/drivers/scsi/libsas/sas_ata.c#L68-L70) 中，`sas_to_ata_err` 函数将 `SAS_ABORTED_TASK` 映射为 `AC_ERR_DEV`

2. **错误标记**：在 [`sas_ata.c`](file:///workspace/drivers/scsi/libsas/sas_ata.c#L138-L150) 中，设置 `qc->err_mask = AC_ERR_DEV`，并设置错误寄存器 `dev->sata_dev.fis[3] = 0x04`，状态寄存器 `dev->sata_dev.fis[2] = ATA_ERR`

3. **错误处理调度**：调用 [`ata_qc_complete(qc)`](file:///workspace/drivers/ata/libata-core.c#L4630-L4723)，检测到 `qc->err_mask` 后设置 `ATA_QCFLAG_FAILED`，然后调用 [`ata_qc_schedule_eh(qc)`](file:///workspace/drivers/ata/libata-eh.c#L891-L920) 调度libata错误处理（**不会立即完成命令**）

4. **libata错误处理**：由 `ata_qc_schedule_eh` 触发libata错误处理流程（libata-eh.c），在错误处理中会：
   - 尝试恢复设备
   - 可能会重置设备
   - 根据配置决定是否重试命令

5. **最终返回SCSI层**：当libata错误处理完成后，通过 [`ata_scsi_qc_complete(qc)`](file:///workspace/drivers/ata/libata-scsi.c#L1641-L1671) 完成处理：
   - 调用 [`ata_gen_ata_sense(qc)`](file:///workspace/drivers/ata/libata-scsi.c#L948-L998) 生成sense数据
   - 设置 `cmd->result = (DRIVER_SENSE << 24) | SAM_STAT_CHECK_CONDITION`
   - sense数据通常为 `ABORTED_COMMAND` 相关的错误

**总结**：
- SATA盘发生 `SAS_ABORTED_TASK` 时，**不会设置 `DID_ABORT`**
- 会触发libata错误处理流程，**可能会重试**（取决于错误处理策略）
- 最终返回SCSI层时，`scmd->result` 为 `SAM_STAT_CHECK_CONDITION`，带有sense数据
- sense数据的具体内容取决于ATA错误寄存器的值，通常为 `ABORTED_COMMAND` sense key

---

## 11. SCSI层对 `DRIVER_SENSE | CHECK_CONDITION` 的处理

### 11.1 处理流程

当 `cmd->result = (DRIVER_SENSE << 24) | SAM_STAT_CHECK_CONDITION` 时，SCSI中层的处理流程如下：

1. **状态检测**：在 [`scsi_error.c`](file:///workspace/drivers/scsi/scsi_error.c#L748-L749) 中，检测到 `CHECK_CONDITION` 状态：

```c
case CHECK_CONDITION:
    return scsi_check_sense(scmd);
```

2. **分析Sense数据**：调用 [`scsi_check_sense`](file:///workspace/drivers/scsi/scsi_error.c#L489-L648) 函数分析sense数据：

```c
int scsi_check_sense(struct scsi_cmnd *scmd)
{
    // 解析sense数据头
    if (!scsi_command_normalize_sense(scmd, &sshdr))
        return FAILED;  /* no valid sense data */
    
    // 报告sense数据
    scsi_report_sense(sdev, &sshdr);
    
    // 处理deferred error
    if (scsi_sense_is_deferred(&sshdr))
        return NEEDS_RETRY;
    
    // 处理ABORTED_COMMAND
    switch (sshdr.sense_key) {
    case ABORTED_COMMAND:
        // 特殊情况处理
        if (sshdr.asc == 0x10) /* DIF */
            return SUCCESS;
        
        if (sshdr.asc == 0x44 && sdev->sdev_bflags & BLIST_RETRY_ITF)
            return ADD_TO_MLQUEUE;
        if (sshdr.asc == 0xc1 && sshdr.ascq == 0x01 &&
            sdev->sdev_bflags & BLIST_RETRY_ASC_C1)
            return ADD_TO_MLQUEUE;
        
        return NEEDS_RETRY;  // 通常会重试
    // 其他sense key处理...
    }
}
```

### 11.2 关键处理逻辑

1. **`DRIVER_SENSE` 的作用**：
   - 表明sense数据是由驱动生成的，而不是设备直接返回的
   - 不影响处理逻辑，只是标识sense数据的来源

2. **`ABORTED_COMMAND` 的处理**：
   - 通常返回 `NEEDS_RETRY`，表示需要重试命令
   - 特殊情况（如DIF错误）返回 `SUCCESS`
   - 特定ASC码可能返回 `ADD_TO_MLQUEUE`（添加到中层队列重新执行）

3. **重试机制**：
   - 当 `scsi_check_sense` 返回 `NEEDS_RETRY` 时，SCSI中层会尝试重试命令
   - 重试次数受 `scsi_cmd_retry_allowed` 限制
   - 重试失败后会进入错误处理流程

### 11.3 与 `DID_ABORT` 的区别

| 状态 | 处理方式 | 重试行为 | 来源 |
|------|---------|---------|------|
| `DID_ABORT` | 直接返回 `SUCCESS`，不分析sense数据 | 不重试 | SAS盘abort |
| `DRIVER_SENSE | CHECK_CONDITION` | 分析sense数据，可能返回 `NEEDS_RETRY` | 可能重试 | SATA盘abort（通过libata） |

### 11.4 实际案例

当SATA盘发生 `SAS_ABORTED_TASK` 时：
1. libata生成 `DRIVER_SENSE | CHECK_CONDITION` 状态
2. SCSI中层调用 `scsi_check_sense` 分析sense数据
3. sense key通常为 `ABORTED_COMMAND`
4. `scsi_check_sense` 返回 `NEEDS_RETRY`
5. SCSI中层尝试重试命令
6. 如果重试失败，进入错误处理流程

**结论**：`DRIVER_SENSE | CHECK_CONDITION` 状态在SCSI中层会被分析sense数据，对于 `ABORTED_COMMAND` 通常会触发重试，这与直接的 `DID_ABORT` 状态处理完全不同。

---

## 12. SATA NCQ错误处理与Read Log机制

### 12.1 NCQ错误事件处理

当SATA设备发生NCQ错误时，PM8001固件会通过`mpi_sata_event`函数上报事件：

```c
// /workspace/drivers/scsi/pm8001/pm80xx_hwi.c:2796-2803
if (event == IO_XFER_ERROR_ABORTED_NCQ_MODE) {
    /* find device using device id */
    pm8001_dev = pm8001_find_dev(pm8001_ha, dev_id);
    /* send read log extension */
    if (pm8001_dev)
        pm80xx_send_read_log(pm8001_ha, pm8001_dev);
    return;
}
```

### 12.2 Read Log命令发送

`pm80xx_send_read_log`函数构造并发送ATA READ LOG EXT命令：

```c
// /workspace/drivers/scsi/pm8001/pm80xx_hwi.c:1814-1891
static void pm80xx_send_read_log(struct pm8001_hba_info *pm8001_ha,
        struct pm8001_device *pm8001_ha_dev)
{
    // ...
    /* construct read log FIS */
    memset(&fis, 0, sizeof(struct host_to_dev_fis));
    fis.fis_type = 0x27;
    fis.flags = 0x80;
    fis.command = ATA_CMD_READ_LOG_EXT;
    fis.lbal = 0x10;  // Read NCQ command error log page
    fis.sector_count = 0x1;
    // ...
    pm8001_ha_dev->id |= NCQ_READ_LOG_FLAG;  // Mark as read log command
    pm8001_ha_dev->id |= NCQ_2ND_RLE_FLAG;
    // ...
}
```

关键要点：
- 读取log page 0x10 (NCQ command error log)
- 设置 `NCQ_READ_LOG_FLAG` 标记标识该命令

### 12.3 Read Log命令完成处理

Read Log命令完成后，在`mpi_sata_completion`函数中处理：

```c
// /workspace/drivers/scsi/pm8001/pm80xx_hwi.c:2486-2496
if (pm8001_dev && (pm8001_dev->id & NCQ_READ_LOG_FLAG)) {
    /* set new bit for abort_all */
    pm8001_dev->id |= NCQ_ABORT_ALL_FLAG;
    /* clear bit for read log */
    pm8001_dev->id = pm8001_dev->id & 0x7FFFFFFF;
    pm80xx_send_abort_all(pm8001_ha, pm8001_dev);
    /* Free the tag */
    pm8001_tag_free(pm8001_ha, tag);
    sas_free_task(t);
    return;
}
```

**重要：Read Log的响应数据没有被保存和处理！** - 驱动只是利用Read Log命令的完成作为触发点，然后直接发送Abort All命令。

### 12.4 SATA事件响应结构

SATA事件响应包含详细的错误信息（`struct sata_event_resp`）：

```c
// /workspace/drivers/scsi/pm8001/pm80xx_hwi.h:571-588
struct sata_event_resp {
    __le32 tag;
    __le32 event;
    __le32 port_id;
    __le32 device_id;
    u32 reserved;
    __le32 event_param0;
    __le32 event_param1;
    __le32 sata_addr_h32;
    __le32 sata_addr_l32;
    __le32 e_udt1_udt0_crc;
    __le32 e_udt5_udt4_udt3_udt2;
    __le32 a_udt1_udt0_crc;
    __le32 a_udt5_udt4_udt3_udt2;
    __le32 hwdevid_diferr;
    __le32 err_framelen_byteoffset;
    __le32 err_dataframe;  // 错误数据帧
} __attribute__((packed, aligned(4)));
```

然而，在`mpi_sata_event`函数中，**这些错误信息并没有被读取和处理**，只是用来判断事件类型。

### 12.5 Abort All命令

Read Log完成后，调用`pm80xx_send_abort_all` abort该设备上的所有命令：

```c
// /workspace/drivers/scsi/pm8001/pm80xx_hwi.c:1764-1811
static void pm80xx_send_abort_all(struct pm8001_hba_info *pm8001_ha,
        struct pm8001_device *pm8001_dev)
{
    // ...
    memset(&task_abort, 0, sizeof(task_abort));
    task_abort.abort_all = cpu_to_le32(1);  // Abort all tasks
    task_abort.device_id = cpu_to_le32(pm8001_ha_dev->device_id);
    task_abort.tag = cpu_to_le32(ccb_tag);
    // ...
}
```

### 12.6 完整流程图

```
SATA NCQ错误发生
    |
    v
固件上报 SATA EVENT 0x23
    |
    v
mpi_sata_event检测到0x23事件
    |
    v
pm80xx_send_read_log (读取log page 0x10)
    |
    v
命令完成，mpi_sata_completion处理
    |
    v
检测到NCQ_READ_LOG_FLAG标记
    |
    v
pm80xx_send_abort_all (Abort所有任务)
    |
    v
libata/SCSI错误处理
    |
    v
设备恢复和重试
```

### 12.7 关键结论

1. **Read Log命令的作用**：不是用来获取错误信息的，而是作为一个流程控制手段
2. **错误信息来源**：固件在事件响应中已经提供了错误信息，但驱动没有处理
3. **Abort All的必要性**：NCQ错误需要清空设备队列，防止错误传播
4. **真正的错误处理**：后续通过libata/SCSI错误恢复机制处理，而不是通过Read Log的数据

---

## 10. SCSI_EH_ABORT_SCHEDULED状态详解

### 10.1 何时设置SCSI_EH_ABORT_SCHEDULED

`SCSI_EH_ABORT_SCHEDULED` 仅在**命令超时时**设置，具体流程如下：

1. **超时检测**：在 [`scsi_error.c`](file:///workspace/drivers/scsi/scsi_error.c#L292-L329) 中，`scsi_times_out` 函数被调用：

```c
enum blk_eh_timer_return scsi_times_out(struct request *req)
{
    struct scsi_cmnd *scmd = blk_mq_rq_to_pdu(req);
    // ...
    if (scsi_abort_command(scmd) != SUCCESS) {
        set_host_byte(scmd, DID_TIME_OUT);
        scsi_eh_scmd_add(scmd);
    }
    // ...
}
```

2. **调度Abort**：调用 [`scsi_abort_command`](file:///workspace/drivers/scsi/scsi_error.c#L194-L221) 设置标志：

```c
static int scsi_abort_command(struct scsi_cmnd *scmd)
{
    // ...
    if (scmd->eh_eflags & SCSI_EH_ABORT_SCHEDULED) {
        // 之前的abort失败，升级到下一层次
        return FAILED;
    }
    // ...
    scmd->eh_eflags |= SCSI_EH_ABORT_SCHEDULED;  // 设置标志
    queue_delayed_work(shost->tmf_work_q, &scmd->abort_work, HZ / 100);
    return SUCCESS;
}
```

### 10.2 SCSI_EH_ABORT_SCHEDULED的作用

1. **标记超时IO**：在 [`scsi_error.c`](file:///workspace/drivers/scsi/scsi_error.c#L1232-L1237) 中，跳过获取sense数据：

```c
/*
 * If SCSI_EH_ABORT_SCHEDULED has been set, it is timeout IO,
 * should not get sense.
 */
list_for_each_entry_safe(scmd, next, work_q, eh_entry) {
    if ((scmd->eh_eflags & SCSI_EH_ABORT_SCHEDULED) ||
        SCSI_SENSE_VALID(scmd))
        continue;
```

2. **统计超时命令**：在 [`scsi_error.c`](file:///workspace/drivers/scsi/scsi_error.c#L374-L377) 中区分失败命令类型：

```c
if (scmd->eh_eflags & SCSI_EH_ABORT_SCHEDULED)
    ++cmd_cancel;
else
    ++cmd_failed;
```

3. **处理DID_ABORT**：在 [`scsi_error.c`](file:///workspace/drivers/scsi/scsi_error.c#L1823-L1828) 中，将超时abort转为 `DID_TIME_OUT`：

```c
case DID_ABORT:
    if (scmd->eh_eflags & SCSI_EH_ABORT_SCHEDULED) {
        set_host_byte(scmd, DID_TIME_OUT);
        return SUCCESS;
    }
    fallthrough;
```

### 10.3 完整的超时处理流程

1. **超时发生**：`scsi_times_out` 被调用
2. **设置标志**：`scmd->eh_eflags |= SCSI_EH_ABORT_SCHEDULED`
3. **调度abort**：`queue_delayed_work` 安排执行abort工作
4. **如果abort失败**：调用 `scsi_eh_scmd_add` 加入错误处理队列
5. **如果abort成功**：命令返回时，`SCSI_EH_ABORT_SCHEDULED` 标志已设置，被转为 `DID_TIME_OUT`

### 10.4 总结

| 情况 | 是否设置SCSI_EH_ABORT_SCHEDULED | 结果 |
|------|--------------------------------|------|
| 命令超时 | 是 | 由超时处理流程处理 |
| 正常IO返回DID_ABORT | 否 | 直接完成，不重试 |
| 错误处理期间abort | 是 | 转为DID_TIME_OUT |
