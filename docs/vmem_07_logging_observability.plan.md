---
name: "VMem 子方案 ⑦ - 日志与可观测性（L3 AI 开发范式）"
overview: "面向 L3 级 AI 开发范式的全链路日志、结构化追踪、运行时健康检查与自诊断体系。核心目标：让 AI 能够自主定位 bug、验证实现正确性、追踪操作全生命周期。"
todos:
  - id: log-infra
    content: "搭建日志基础设施：Serilog + 结构化 JSON 输出 + 多 Sink 配置"
    status: pending
  - id: log-correlation
    content: "实现操作关联追踪：CorrelationId 贯穿 WinFsp 回调→内存引擎→IPC 全链路"
    status: pending
  - id: log-contracts
    content: "实现契约式编程：前置/后置条件断言 + 不变量检查（Release 也生效）"
    status: pending
  - id: log-health
    content: "实现健康检查端点：PagePool/DirectoryTree/Service 状态可查询"
    status: pending
  - id: log-ai-digest
    content: "实现 AI 摘要日志：每次操作/每个 Phase 完成后输出机器可读摘要"
    status: pending
  - id: log-replay
    content: "实现操作回放日志：记录足够信息支持 bug 场景离线复现"
    status: pending
  - id: log-perf
    content: "实现性能追踪：关键路径耗时计量 + 慢操作自动告警"
    status: pending
  - id: log-rotation
    content: "实现日志轮转与归档：按大小/时间轮转 + 压缩归档"
    status: pending
isProject: false
---

# 子方案 ⑦ - 日志与可观测性（L3 AI 开发范式）

> 父方案：[VMem RAM Disk Design](./vmem_ram_disk_design_031231c8.plan.md)
> 对应代码：`src/VMem.Core/Diagnostics/` + 各模块内嵌日志
> Phase：**全程**（与 Phase 1 同步启动，贯穿所有阶段）

---

## 0. 什么是 L3 级 AI 开发范式？

| 级别 | 模式 | 人类角色 | AI 角色 | 对日志的要求 |
|------|------|---------|---------|-------------|
| L1 | AI 辅助 | 编码者 | 补全/建议 | 人类读日志 |
| L2 | AI 协作 | 设计者+审查者 | 实现者 | 人类和 AI 都读日志 |
| **L3** | **AI 主导** | **监督者** | **全流程执行者** | **AI 必须能自主读懂、诊断、定位日志** |

**L3 范式对日志系统的核心要求**：

1. **结构化**：JSON 格式，字段固定、可机器解析，AI 不需要正则表达式就能提取信息
2. **关联性**：一次用户操作从 GUI → IPC → FS 回调 → 内存引擎，全链路可追踪
3. **自诊断**：系统定期自检并输出健康报告，AI 无需猜测内部状态
4. **契约化**：每个关键方法有前置/后置条件，违反时立即记录上下文
5. **可回放**：记录的信息足够让 AI 离线复现问题场景
6. **分级告警**：异常不只是记录，还要主动标记严重程度供 AI 优先处理

---

## 1. 日志基础设施

### 1.1 框架选型：Serilog

| 选项 | 结构化 | 性能 | .NET 9 | AOT | 选择理由 |
|------|--------|------|--------|-----|---------|
| Microsoft.Extensions.Logging | 半结构化 | 好 | ✓ | ✓ | 仅作抽象层 |
| **Serilog** | **原生结构化** | **优秀** | **✓** | **✓** | 最成熟的 .NET 结构化日志库 |
| NLog | 结构化 | 好 | ✓ | 部分 | 社区活跃度略低 |

**最终方案**：`Microsoft.Extensions.Logging` 作为抽象接口 + `Serilog` 作为实现后端。

### 1.2 日志 Sink（输出目标）

| Sink | 格式 | 用途 | V1 启用 | V2+ |
|------|------|------|---------|------|
| Console (Stdout) | 彩色文本 | 开发调试、CLI 模式 | ✓ | ✓ |
| **File (JSON)** | 结构化 JSON，一行一条 | **AI 解析的主要数据源** | **✓（单 Sink）** | ✓ |
| File (Text) | 人类可读文本 | 人类排错 | ✗（V2） | ✓ |
| Windows EventLog | 系统事件 | Service 故障排查 | ✗（V2） | ✓ |
| Seq / OpenTelemetry | OTLP | 未来扩展 | ✗ | 可选 |

> **辩论裁决 R2**：V1 双 File Sink（JSON + Text）是审查 P0 性能问题。V1 仅保留单 JSON Sink。
> 热路径 INF 日志也在 V1 降为 Warning 默认级别。

### 1.3 JSON 日志格式规范

每条日志是一行 JSON，字段固定，AI 可直接 `jq` 或代码解析：

```json
{
  "ts": "2026-06-16T00:05:32.1234567+08:00",
  "level": "INF",
  "msg": "Write completed",
  "correlationId": "a1b2c3d4",
  "source": "PagedFileContent",
  "method": "Write",
  "props": {
    "filePath": "\\Temp\\data.bin",
    "offset": 0,
    "length": 65536,
    "pagesAllocated": 16,
    "phase1_scanMs": 0.02,
    "phase2_allocMs": 0.15,
    "phase3_writeMs": 0.08,
    "totalMs": 0.25,
    "poolAvailablePages": 245760
  }
}
```

**固定字段约定**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `ts` | ISO 8601 | 精确到 100ns（与 FILETIME 对齐） |
| `level` | string | `VRB` / `DBG` / `INF` / `WRN` / `ERR` / `FTL` |
| `msg` | string | 简短的操作描述（英文，便于 AI 匹配模式） |
| `correlationId` | string | 8 位十六进制，贯穿单次 FS 操作全链路 |
| `source` | string | 发出日志的类名 |
| `method` | string | 发出日志的方法名 |
| `props` | object | **可变属性包**，按操作类型不同携带不同字段 |
| `error` | object? | 异常时携带 `{ type, message, stackTrace }` |
| `contract` | object? | 契约违反时携带 `{ name, expected, actual }` |

---

## 2. 日志级别规范

| 级别 | 使用场景 | 示例 | AI 处理策略 |
|------|---------|------|------------|
| `VRB` (Verbose) | 逐页/逐扇区级细节 | "Page 0x7FF allocated at index 42" | 仅在复现 bug 时开启 |
| `DBG` (Debug) | 方法入口/出口、中间状态 | "Write Phase 1: 3 missing pages found" | 开发时默认开启 |
| `INF` (Information) | **关键业务操作完成** | "Disk R: mounted (2GB, 4KB pages)" | **AI 主要关注层** |
| `WRN` (Warning) | 非致命异常、性能退化 | "Pool utilization >90%, 245 pages left" | AI 标记为待观察 |
| `ERR` (Error) | 操作失败但系统仍可用 | "Write failed: STATUS_DISK_FULL" | **AI 立即分析** |
| `FTL` (Fatal) | 系统不可恢复 | "PagePool disposed with 128 leaked pages" | **AI 立即停止并报告** |

### 默认级别配置（辩论裁决 R2：V1 默认 Warning）

| 场景 | 控制台 | JSON 文件 | 文本文件(V2) |
|------|--------|----------|-------------|
| 开发 (Debug) | `DBG` | `DBG` | — |
| **V1 生产 (Release)** | **`WRN`** | **`WRN`** | — |
| V2 生产 (Release) | `INF` | `INF` | `INF` |
| 排错 (Troubleshoot) | `DBG` | `DBG` | — |

> V1 默认 Warning 是性能与诊断的折中。通过 IPC/CLI `vmem log-level Debug` 可运行时热切换。

---

## 3. 操作关联追踪（CorrelationId）

### 3.1 设计

每次 WinFsp 回调入口生成一个 `CorrelationId`（8 位 hex），通过 **`[ThreadStatic]`** 在整个调用链中传播：

> **辩论裁决 R2/R10**：WinFsp 回调来自非托管线程池，`AsyncLocal` 在非 async 上下文可能丢失。
> 改用 `[ThreadStatic]` + 显式 `BeginFsOperation` scope，确保可靠传播。

```
WinFsp 回调入口 (VMemFileSystem.Write)
    │ correlationId = "a1b2c3d4"
    ▼
PagedFileContent.Write()          ← 自动携带同一 correlationId
    │
    ├─ Phase 1: ReadLock scan     ← log: correlationId=a1b2c3d4
    ├─ Phase 2: PagePool.TryRent  ← log: correlationId=a1b2c3d4
    └─ Phase 3: WriteLock memcpy  ← log: correlationId=a1b2c3d4
```

### 3.2 实现

```csharp
public static class OperationContext
{
    [ThreadStatic] private static string? t_correlationId;

    public static string CorrelationId
    {
        get => t_correlationId ?? "--------";
        set => t_correlationId = value;
    }

    public static IDisposable BeginFsOperation(string callbackName)
    {
        var id = Guid.NewGuid().ToString("N")[..8];
        t_correlationId = id;
        // 通过 Serilog LogContext 注入，后续日志自动携带
        return new OperationScope(id, callbackName);
    }

    private sealed class OperationScope : IDisposable
    {
        private readonly string _id;
        public OperationScope(string id, string callback)
        {
            _id = id;
            // Push to Serilog LogContext for enrichment
        }
        public void Dispose() => t_correlationId = null;
    }
}
```

> **IPC 关联**：IPC `RequestId` 进入 Service 后直接赋值 `OperationContext.CorrelationId = request.RequestId`。
> FS 子操作可派生 `{RequestId}.{seq}`。

### 3.3 跨进程关联（IPC）

GUI → Service 的 IPC 请求中携带 `RequestId`，Service 端将其映射为 `CorrelationId`，实现跨进程追踪：

```json
{ "type": "CreateDisk", "requestId": "gui-5678", "driveLetter": "R:", ... }
```

Service 日志：`correlationId=gui-5678`，AI 可关联 GUI 和 Service 两端的日志。

---

## 4. 契约式编程（Contract Checking）

### 4.1 设计理念

不同于传统 `Debug.Assert`（Release 构建中被移除），L3 范式要求**关键不变量在生产环境也检查**，违反时记录完整上下文而非崩溃：

```csharp
public static class Contract
{
    /// 前置条件：方法入口检查参数合法性
    public static void Requires(bool condition, string name,
        [CallerMemberName] string? caller = null,
        [CallerFilePath] string? file = null,
        [CallerLineNumber] int line = 0)
    {
        if (!condition)
        {
            Log.Error("CONTRACT VIOLATION [Requires] {Name} in {Caller} at {File}:{Line}",
                name, caller, file, line);
            // Release: 记录日志 + 返回错误码，不崩溃
            // Debug: 额外触发 Debugger.Break()
        }
    }

    /// 后置条件：方法出口检查返回值/状态合法性
    public static void Ensures(bool condition, string name, object? context = null, ...);

    /// 不变量：检查对象/系统级不变量
    public static void Invariant(bool condition, string name, object? context = null, ...);
}
```

### 4.2 关键契约清单

| 模块 | 契约名 | 检查时机 | 表达式 |
|------|--------|---------|--------|
| PagePool | `PoolConsistent` | 每次 Rent/Return 后 | `rentedCount + freeStack.Count <= allocatedCount` |
| PagePool | `NoNegativeCount` | 每次 Return 后 | `rentedCount >= 0 && reservedCount >= 0` |
| PagePool | `CapacityNotExceeded` | 每次 Rent/Reserve 后 | `rentedCount + reservedCount <= maxPages` |
| PagedFileContent | `PageTableIntegrity` | Write Phase 3 后 | 所有非零页表项指向有效的 NativeMemory 地址 |
| PagedFileContent | `FileSizeConsistent` | SetLength 后 | `fileSize >= 0 && fileSize <= maxPages * pageSize` |
| DirectoryTree | `NoOrphanNodes` | Remove 后 | 被删节点不在任何父目录的 Children 中 |
| VMemFileSystem | `NotifyAfterMutation` | Rename/Delete/Overwrite 后 | `FspNotify` 已被调用（EnableKernelCache 时） |

### 4.3 契约违反日志格式

```json
{
  "ts": "...",
  "level": "ERR",
  "msg": "CONTRACT VIOLATION",
  "contract": {
    "type": "Invariant",
    "name": "PoolConsistent",
    "expected": "rentedCount(500) + freeCount(499) <= allocatedCount(1000)",
    "actual": "rentedCount(500) + freeCount(502) = 1002 > allocatedCount(1000)",
    "caller": "PagePool.Return",
    "file": "PagePool.cs",
    "line": 87
  },
  "correlationId": "a1b2c3d4"
}
```

AI 看到此日志后可直接定位：`PagePool.Return` 第 87 行的归还逻辑导致 free count 多了 2，可能是重复归还。

---

## 5. 健康检查与自诊断

### 5.1 HealthCheck 服务（辩论裁决 R2：V1 按需，非定时）

> V1 仅在 **Dispose / 卸载盘 / IPC `GetStatus`** 时按需执行，输出一条 JSON。
> V2 再做 30s 后台定时器。

```csharp
public sealed class HealthChecker
{
    public HealthReport Check()
    {
        return new HealthReport(
            Timestamp: DateTime.UtcNow,
            Components: [
                CheckPagePool(),       // 页池一致性
                CheckDirectoryTree(),  // 目录树完整性
                CheckServiceConnection(), // IPC 连接状态
                CheckSystemMemory(),   // 系统物理内存
                CheckWinFspDriver(),   // WinFsp 驱动状态
            ]);
    }
}
```

### 5.2 健康报告日志格式

```json
{
  "ts": "...",
  "level": "INF",
  "msg": "Health check completed",
  "source": "HealthChecker",
  "props": {
    "overall": "Healthy",
    "components": {
      "PagePool": { "status": "Healthy", "rented": 500, "reserved": 100, "available": 400, "leakSuspect": false },
      "DirectoryTree": { "status": "Healthy", "totalNodes": 1234, "maxDepth": 8 },
      "IPC": { "status": "Healthy", "connectedClients": 1 },
      "SystemMemory": { "status": "Warning", "availableGB": 2.1, "totalGB": 32.0, "usedPercent": 93.4 },
      "WinFsp": { "status": "Healthy", "version": "2.0.23075" }
    }
  }
}
```

AI 定期检查此日志，`Warning` 或 `Unhealthy` 状态立即触发分析。

---

## 6. AI 摘要日志（Digest Log）

### 6.1 操作级摘要

每个 WinFsp 回调完成后，输出一行包含完整上下文的摘要日志，AI 只需扫描 `level=INF` 的日志即可了解系统所有活动：

```json
{
  "level": "INF",
  "msg": "FS.Write OK",
  "correlationId": "a1b2c3d4",
  "props": {
    "path": "\\Temp\\data.bin",
    "offset": 0,
    "requestedBytes": 65536,
    "writtenBytes": 65536,
    "pagesAllocated": 16,
    "totalMs": 0.25,
    "result": "STATUS_SUCCESS"
  }
}
```

### 6.2 周期性统计摘要

每 60 秒输出一次系统级统计摘要：

```json
{
  "level": "INF",
  "msg": "Periodic stats digest",
  "source": "StatsCollector",
  "props": {
    "intervalSec": 60,
    "totalReadMB": 1024.5,
    "totalWriteMB": 512.3,
    "readOps": 15000,
    "writeOps": 8000,
    "avgReadLatencyUs": 12.5,
    "avgWriteLatencyUs": 45.2,
    "peakReadMBps": 3200.0,
    "peakWriteMBps": 1800.0,
    "poolUtilization": 0.62,
    "fileCount": 1234,
    "contractViolations": 0
  }
}
```

---

## 7. 操作回放日志（Replay Log）

### 7.1 目的

当 AI 发现 bug 时，能够**离线复现**问题序列，而无需实际挂载 RAM 盘。

### 7.2 格式

当日志级别设为 `VRB` 时，记录每个 FS 操作的完整输入输出，可用于编写回归测试：

```json
{
  "level": "VRB",
  "msg": "FS.Replay",
  "props": {
    "seq": 42,
    "op": "Write",
    "input": { "path": "\\a.txt", "offset": 0, "length": 100, "dataHash": "sha256:abc..." },
    "output": { "status": "STATUS_SUCCESS", "bytesWritten": 100 },
    "preState": { "fileSize": 0, "allocatedPages": 0 },
    "postState": { "fileSize": 100, "allocatedPages": 1 }
  }
}
```

AI 可将此序列转化为自动化测试用例：

```csharp
// Auto-generated regression test from replay log
[Fact]
public void Replay_Seq42_Write_a_txt()
{
    var fs = CreateTestFileSystem();
    fs.Create("\\a.txt", ...);
    var result = fs.Write("\\a.txt", offset: 0, data: new byte[100]);
    Assert.Equal(NtStatus.STATUS_SUCCESS, result);
    Assert.Equal(100, fs.GetFileSize("\\a.txt"));
}
```

---

## 8. 性能追踪

### 8.1 关键路径计时

使用 `Stopwatch` 对热路径进行微秒级计时：

| 路径 | 计时点 | 告警阈值 |
|------|--------|---------|
| Write 三阶段 | Phase1 / Phase2 / Phase3 分别计时 | 单次 Write > 10ms |
| Read | 总耗时 | 单次 Read > 5ms |
| PagePool.TryRent | 分配耗时 | 单次 > 1ms（可能触发了 OS 页错误） |
| Rename | 加锁等待 + 执行 | 加锁等待 > 50ms（可能死锁） |
| IPC 往返 | 请求→响应 | > 500ms |
| Snapshot | 增量写入 | > 10s |

### 8.2 慢操作自动告警

```json
{
  "level": "WRN",
  "msg": "SLOW OPERATION",
  "source": "PagedFileContent",
  "method": "Write",
  "props": {
    "totalMs": 15.3,
    "threshold": 10.0,
    "breakdown": {
      "phase1_scanMs": 0.5,
      "phase2_allocMs": 14.2,
      "phase3_writeMs": 0.6
    },
    "diagnosis": "Phase 2 allocation took 14.2ms, likely OS page fault during NativeMemory.AllocZeroed"
  }
}
```

---

## 9. 日志轮转与归档

### 9.1 策略

| 日志类型 | 文件路径 | 轮转策略 | 保留 |
|---------|---------|---------|------|
| JSON 日志 | `%ProgramData%\VMem\logs\vmem-YYYYMMDD.json` | 每天 / 100MB | 7 天 |
| 文本日志 | `%ProgramData%\VMem\logs\vmem-YYYYMMDD.log` | 每天 / 100MB | 7 天 |
| 回放日志 | `%ProgramData%\VMem\logs\replay-YYYYMMDD.jsonl` | 每天 / 500MB | 3 天 |
| 归档 | `%ProgramData%\VMem\logs\archive\` | gzip 压缩 | 30 天 |

### 9.2 配置

```jsonc
{
  "Logging": {
    "MinimumLevel": "Information",      // 可通过 IPC 运行时修改
    "EnableReplayLog": false,           // 排错时开启
    "JsonLogPath": "%ProgramData%\\VMem\\logs",
    "RetentionDays": 7,
    "MaxFileSizeMB": 100
  }
}
```

---

## 10. 项目代码结构

```
src/VMem.Core/Diagnostics/
├── VMemLogger.cs            # Serilog 初始化 + 配置
├── OperationContext.cs       # CorrelationId + AsyncLocal
├── Contract.cs              # 前置/后置/不变量检查
├── HealthChecker.cs         # 周期性健康检查
├── SlowOperationDetector.cs # 慢操作告警
└── DigestWriter.cs          # 周期性统计摘要输出
```

---

## 11. 各子方案的 L3 范式补丁清单

经审计，现有子方案 ①~⑥ 存在以下不符合 L3 AI 开发范式的缺口：

### 子方案 ① 内存引擎

| 缺口 | 补丁 |
|------|------|
| PagePool 操作无日志 | 每次 Rent/Return/Reserve 在 `DBG` 级别记录，附带 correlationId 和池状态快照 |
| 泄漏检测仅 Debug 断言 | 改为 `Contract.Invariant` + `ERR` 日志，Release 也生效 |
| 无性能计时 | TryRent 内嵌 Stopwatch，> 1ms 输出 `WRN` |

### 子方案 ② 文件系统

| 缺口 | 补丁 |
|------|------|
| WinFsp 回调无关联 ID | 每个回调入口调用 `OperationContext.BeginOperation()`，出口输出摘要日志 |
| 三阶段写入无分阶段计时 | Phase 1/2/3 分别计时，慢操作自动分析瓶颈阶段 |
| Notify 调用无日志 | 每次 `FspNotify` 调用记录路径和操作类型 |
| DirectoryTree 无操作审计 | Create/Rename/Delete 在 `INF` 级别记录路径和结果 |
| 契约检查仅在 Debug | `PageTableIntegrity`、`NotifyAfterMutation` 改为 Release 也检查 |

### 子方案 ③ IPC 与服务化

| 缺口 | 补丁 |
|------|------|
| IPC 请求无追踪 | 每个请求携带 `requestId`，映射为 Service 端 correlationId |
| Service 启停无日志 | 启动/停止/自动挂载恢复在 `INF` 记录完整参数 |
| 无运行时日志级别切换 | 新增 `SetLogLevel` IPC 命令，AI 可远程调整日志详细度 |
| 配置变更无审计 | ConfigStore 每次保存记录 diff（旧值 vs 新值） |

### 子方案 ④ GUI

| 缺口 | 补丁 |
|------|------|
| GUI 异常无结构化上报 | 全局异常处理器捕获未处理异常，写入 JSON 日志 |
| 用户操作无审计 | 关键操作（创建/卸载/设置变更）记录到日志 |

### 子方案 ⑤ 持久化与部署

| 缺口 | 补丁 |
|------|------|
| 快照过程无进度日志 | 快照开始/脏页数/写入字节数/耗时/成败 在 `INF` 记录 |
| 恢复过程无校验日志 | Restore 时逐步记录：Header 校验/页恢复数/目录树恢复 |

### 子方案 ⑥ 测试策略

| 缺口 | 补丁 |
|------|------|
| 无日志驱动的自动化回归 | 新增：从回放日志自动生成回归测试用例 |
| CI 无日志归档 | GitHub Actions 产物（Artifacts）中上传测试运行的 JSON 日志 |
| 无契约违反统计 | CI 中解析 JSON 日志，`CONTRACT VIOLATION` 计数 > 0 则 fail |

---

## 12. 依赖

| 包 | 版本 | 用途 |
|----|------|------|
| Serilog | 4.x | 结构化日志核心 |
| Serilog.Sinks.Console | 6.x | 彩色控制台输出 |
| Serilog.Sinks.File | 6.x | JSON + Text 文件输出 |
| Serilog.Sinks.EventLog | 4.x | Windows EventLog (Service) |
| Serilog.Extensions.Hosting | 8.x | 与 Microsoft.Extensions.Hosting 集成 |
| Serilog.Formatting.Compact | 3.x | 紧凑 JSON 格式 |
