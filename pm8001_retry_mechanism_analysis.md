# PM8001驱动IO错误重试机制分析

## 1. 概述

本文档分析Linux内核5.10版本中PM8001 SAS/SATA HBA驱动在IO错误发生时的重试机制，涵盖了SCSI层、libsas层和PM8001驱动层三个层面的处理逻辑。

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
