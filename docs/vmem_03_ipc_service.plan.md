---
name: "VMem 子方案 ③ - IPC 与服务化"
overview: "Named Pipe 双通道通信协议、Windows Service 开机自启、双进程模型（Service 承载 FS，GUI 远程控制）。"
todos:
  - id: ipc-protocol
    content: "定义 IPC 消息协议（record 类型 + JSON 序列化）"
    status: pending
  - id: pipe-server
    content: "实现 PipeServer：请求-响应 + PipeSecurity 权限控制"
    status: pending
  - id: pipe-client
    content: "实现 PipeClient：连接/重连/超时处理"
    status: pending
  - id: event-pipe
    content: "实现事件推送管道（Service → GUI 单向流）"
    status: pending
  - id: win-service
    content: "实现 Windows Service（BackgroundService + 自动挂载）"
    status: pending
  - id: config-store
    content: "实现 ConfigStore：JSON 持久化自动挂载列表"
    status: pending
  - id: cli-tool
    content: "实现 CLI 工具：vmem mount/unmount/status"
    status: pending
isProject: false
---

# 子方案 ③ - IPC 与服务化

> 父方案：[VMem RAM Disk Design](./vmem_ram_disk_design_031231c8.plan.md)
> 对应代码：`src/VMem.Core/Ipc/` + `src/VMem.Core/Service/` + `src/VMem.Service/`
> Phase：1（CLI）、3（Service + IPC）

---

## 1. 进程模型

| 进程 | 项目 | 运行身份 | 职责 | Phase |
|------|------|----------|------|-------|
| vmem.exe | VMem.Cli | 当前用户 | CLI 前台挂载/查询，直接调用 Core | **1** |
| VMem.Service.exe | VMem.Service | LocalSystem | 承载 WinFsp FS，管理 RAM 盘生命周期，开机自启 | 3 |
| VMem.App.exe | VMem.App | 当前用户 | WPF GUI，通过 Named Pipe 远程控制 Service | 2 |

**关键决策**：FS 运行在 Service 进程中 → GUI 崩溃不影响已挂载磁盘。

---

## 1.1 CLI 工具设计（Phase 1，辩论裁决 R13/R44-R46）

```
vmem mount <drive> [options]     # 前台阻塞挂载
vmem status [drive]              # 查看磁盘状态
vmem unmount <drive>             # 卸载（Phase 3 通过 IPC）
vmem version                     # 版本信息
```

### mount 子命令

```
vmem mount R: --size 2G [--page-size 4|64] [--cache safe|balanced|fast]
              [--label "My Disk"] [--sddl "D:P(A;;FA;;;IU)"]
              [--pre-allocate] [--verbose]
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `<drive>` | 必需 | 盘符（A:-Z:） |
| `--size` | 必需 | 容量（支持 K/M/G/T 后缀） |
| `--page-size` | 4 | 页大小 KB |
| `--cache` | safe | 缓存模式 |
| `--label` | "VMem" | 卷标 |
| `--sddl` | IU 默认 | Root 安全描述符 |
| `--pre-allocate` | false | 预分配所有页面 |
| `--verbose` | false | 日志级别切换到 Debug |

**运行模式**：
- 前台阻塞：`Console.CancelKeyPress` 捕获 Ctrl+C → 优雅卸载（StopDispatcher → Dispose）
- 退出码：VmErrorCode 映射（0=成功, 1=DiskFull, 2=DriveInUse, 3=InvalidArg, 4=Internal, 130=Ctrl+C）

### status 子命令

```
vmem status          # 列出所有磁盘
vmem status R:       # 指定磁盘详情
vmem status --format json   # JSON 输出（管道友好）
```

- TTY 检测：`Console.IsOutputRedirected` → true 时自动 JSON 输出
- 表格格式：Drive | Label | Size | Used | Free | Files | Status

### Phase 1 CLI 架构

```csharp
// Program.cs
var root = new RootCommand("VMem RAM Disk");
root.AddCommand(new MountCommand());
root.AddCommand(new StatusCommand());
return await root.InvokeAsync(args);

// MountCommand.cs
internal sealed class MountCommand : Command
{
    public MountCommand() : base("mount", "Mount a RAM disk") { /* options */ }

    public async Task<int> HandleAsync(/* ... */, CancellationToken ct)
    {
        // Amazon 审查：完整的异常处理 + 参数校验 + 优雅退出
        var config = DiskConfig.FromCliArgs(drive, size, pageSize, cache, label, sddl, preAllocate);
        if (config.CapacityBytes < 1024 * 1024) // 最小 1MB
        {
            Console.Error.WriteLine("Error: minimum size is 1MB");
            return (int)VmErrorCode.InvalidArgument;
        }

        using var manager = new RamDiskManager();
        try
        {
            await manager.MountAsync(config, ct);
            Console.WriteLine($"Mounted {config.DriveLetter} ({config.Label}, " +
                $"{config.CapacityBytes / 1024 / 1024}MB, cache={config.CacheMode})");
            Console.WriteLine("Press Ctrl+C to unmount and exit.");

            var tcs = new TaskCompletionSource();
            Console.CancelKeyPress += (_, e) => { e.Cancel = true; tcs.SetResult(); };
            await tcs.Task;

            Console.WriteLine("Unmounting...");
            await manager.UnmountAsync(config.DriveLetter, CancellationToken.None);
            Console.WriteLine("Unmounted successfully.");
            return 0;
        }
        catch (DriveInUseException)
        {
            Console.Error.WriteLine($"Error: drive {config.DriveLetter} is already in use");
            return (int)VmErrorCode.DriveInUse;
        }
        catch (OutOfMemoryException)
        {
            Console.Error.WriteLine("Error: not enough memory to create RAM disk");
            return (int)VmErrorCode.DiskFull;
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            Console.Error.WriteLine($"Fatal: {ex.Message}");
            _logger.Fatal(ex, "CLI mount failed");
            return (int)VmErrorCode.Internal;
        }
    }
}
```

---

## 2. Named Pipe IPC

### 2.1 控制通道 + IPC 安全基线（辩论裁决 R5）

```
管道名: \\.\pipe\vmem-control-v1
协议:   请求-响应，4字节 BE 长度前缀 + JSON（辩论裁决 R15-3）
```

**三层认证（无 HMAC）**：

| 层级 | 机制 | 说明 |
|------|------|------|
| L1 连接 | PipeSecurity ACL | 拒绝 Everyone；允许 BA + Interactive User SID |
| L2 请求 | ImpersonateNamedPipeClient | **写操作必做** Token 校验；读操作可缓存 SID |
| L3 资源 | CreatorSid 归属 | CreateDisk 写入 CreatorSid；Remove/Update 校验 IsAdmin \|\| callerSid == CreatorSid |

**流量限制**：

| 参数 | 值 | 说明 |
|------|-----|------|
| 单条消息上限 | **64 KB** | 读前检查，超限断开 |
| 全局 per-connection | **20 req/s，burst 40** | 正常 GUI 绰绰有余 |
| 写操作 per-connection | **2 req/s，burst 5** | GUI debounce 300ms 天然合规 |
| 并发连接 | **8** | 多用户 RDP 够用 |
| 超限行为 | 返回 `IpcResponse(RateLimited)`，连续 10 次超限才断开 | |

### 2.2 消息协议（辩论裁决 R15：长度前缀 + DiskConfig 引用）

**Wire 分帧**（R15-3）：`[4字节 BE 长度][JSON UTF-8 body]`，body ≤ 64KB。废弃换行分隔。

```csharp
[JsonPolymorphic(TypeDiscriminatorPropertyName = "type")]
[JsonDerivedType(typeof(CreateDiskRequest), "CreateDisk")]
[JsonDerivedType(typeof(RemoveDiskRequest), "RemoveDisk")]
// ... 其他 derived types
public record IpcMessage(string Type)
{
    public string RequestId { get; init; } = "";
}

// ─── 磁盘生命周期（R15-1：引用 DiskConfig 而非扁平 11 字段）───
public record ClientDiskConfig  // IPC 入站 DTO，无 CreatorSid
{
    public required string DriveLetter { get; init; }
    public required long CapacityBytes { get; init; }
    public string Label { get; init; } = "VMem RAM Disk";
    public DiskPerformanceOptions Performance { get; init; } = new();
    public DiskSecurityOptions Security { get; init; } = new();
    public DiskSnapshotOptions Snapshot { get; init; } = new();
}

public record CreateDiskRequest(
    ClientDiskConfig Config,
    bool AddToAutoMount
) : IpcMessage("CreateDisk");
// Service 处理时：config.ToDiskConfig(ctx.CallerSid) → MountAsync

public record RemoveDiskRequest(
    string DriveLetter,
    bool RemoveFromAutoMount            // ← 同时从自动挂载列表移除？
) : IpcMessage("RemoveDisk");

// ─── 磁盘配置编辑 ───
public record UpdateDiskConfigRequest(
    string DriveLetter,
    string? NewLabel,                   // null = 不修改
    bool? AutoMount,
    bool? EnableSnapshot,
    int? SnapshotIntervalMinutes
) : IpcMessage("UpdateDiskConfig");
// 注意：容量、页大小、内核缓存 等需要重新挂载才能生效，不支持热修改

// ─── 自动挂载开关 ───
public record ToggleAutoMountRequest(
    string DriveLetter,
    bool Enable                         // true=加入自动挂载, false=移除
) : IpcMessage("ToggleAutoMount");

// ─── 查询 ───
public record QueryStatusRequest() : IpcMessage("QueryStatus");
public record QueryStatsRequest(string DriveLetter) : IpcMessage("QueryStats");
public record GetConfigRequest() : IpcMessage("GetConfig");       // 获取完整配置

// ─── 全局设置 ───
public record UpdateGlobalSettingsRequest(
    int? MemoryWarningPercent,          // 系统内存告警阈值 (默认 90)
    string? LogLevel                    // 运行时日志级别切换
) : IpcMessage("UpdateGlobalSettings");

// ─── 响应（辩论裁决 R15-2：VmErrorCode 替换 bool+string）───
public record IpcResponse(VmErrorCode Code, string? ErrorDetail = null)
{
    [JsonIgnore] public bool Success => Code == VmErrorCode.Success;
    public object? Data { get; init; }
}
// VmErrorCode 新增：ServiceUnavailable = 7

public record StatusResponse(List<DiskInfo> Disks) : IpcResponse(VmErrorCode.Success);

public record DiskInfo(
    string DriveLetter, string Label, long CapacityBytes,
    long UsedBytes, long ReservedBytes, int FileCount,
    bool AutoMount, bool EnableSnapshot,
    string State  // "Mounted" | "Mounting" | "Unmounting" | "Error"
);

public record StatsResponse(long UsedBytes, long TotalBytes, long ReservedBytes,
    int FileCount, double ReadBytesPerSec, double WriteBytesPerSec,
    double ReadOpsPerSec, double WriteOpsPerSec) : IpcResponse(VmErrorCode.Success);

public record ConfigResponse(
    GlobalSettings Global,
    List<DiskConfig> AutoMountDisks
) : IpcResponse(VmErrorCode.Success);
```

### 2.3 PipeServer 实现细节（辩论裁决 R16）

| 参数 | 值 | 说明 |
|------|-----|------|
| Accept 模型 | 循环 Accept + 每连接 Task | SemaphoreSlim(8) 限并发 |
| PipeDirection | Control=InOut；Events=Out(Server)/In(Client) | Byte 模式 |
| 身份校验 | Connect 缓存 SID；写操作 per-request Impersonate | 读可用缓存 SID |
| 断开检测 | Read 返回 0 + IOException | 120s 空闲超时；V1 无 KeepAlive |

**PipeClient 重连策略**：

| 策略 | 参数 |
|------|------|
| 后台无限重连 | 500ms→30s 指数退避 |
| RPC 调用重试 | Polly 3次：200ms / 500ms / 1500ms |
| 状态通知 | 断连 → GUI 显示黄色断连条（辩论裁决 R5） |

```csharp
// PipeFraming 工具类（辩论裁决 R15-3）
public static class PipeFraming
{
    // Google 审查：避免 header.ToArray() 堆分配
    public static async Task WriteMessageAsync(PipeStream pipe, ReadOnlyMemory<byte> json, CancellationToken ct)
    {
        if (json.Length > MaxMessageSize) throw new ArgumentException($"Message too large: {json.Length}");
        var header = new byte[4]; // 栈上 4 字节可接受；或复用线程本地缓冲
        BinaryPrimitives.WriteInt32BigEndian(header, json.Length);
        await pipe.WriteAsync(header, ct);
        await pipe.WriteAsync(json, ct);
        await pipe.FlushAsync(ct); // Amazon 审查：确保对端立即收到
    }

    private const int MaxMessageSize = 65536; // 64KB

    public static async Task<byte[]?> ReadMessageAsync(PipeStream pipe, int maxSize, CancellationToken ct)
    {
        var header = new byte[4];
        if (await ReadExactAsync(pipe, header, ct) != 4) return null; // 断开
        var len = BinaryPrimitives.ReadInt32BigEndian(header);
        if (len <= 0 || len > maxSize) throw new ProtocolViolationException($"msg size {len}");
        var body = new byte[len];
        if (await ReadExactAsync(pipe, body, ct) != len) return null;
        return body;
    }
}
```

### 2.4 事件推送通道

```
管道名: \\.\pipe\vmem-events-v1
方向:   Service → GUI（单向流）
```

| 事件 | 触发时机 |
|------|---------|
| `DiskMounted` / `DiskUnmounted` | 磁盘状态变更 |
| `DiskCreatingProgress` | PreAllocate 模式下创建进度（Percent, AllocatedMB, TotalMB） |
| `MemoryWarning` | 系统可用物理内存使用率超过阈值 |
| `SnapshotCompleted` / `SnapshotFailed` | 快照完成/失败 |
| `ServiceError` | Service 内部未预期错误（非致命） |

---

## 3. Windows Service

### 3.0 RamDiskManager 编排器（辩论裁决 R14）

```csharp
public sealed class RamDiskManager : IAsyncDisposable
{
    private readonly ConcurrentDictionary<string, MountedDisk> _disks = new(StringComparer.OrdinalIgnoreCase);
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _mountLocks = new(StringComparer.OrdinalIgnoreCase);

    public async Task MountAsync(DiskConfig config, CancellationToken ct)
    {
        // Amazon 审查：参数校验前置
        ArgumentException.ThrowIfNullOrWhiteSpace(config.DriveLetter);
        if (config.CapacityBytes <= 0) throw new ArgumentOutOfRangeException(nameof(config.CapacityBytes));
        if (_disks.ContainsKey(config.DriveLetter))
            throw new DriveInUseException(config.DriveLetter);

        var sem = _mountLocks.GetOrAdd(config.DriveLetter, _ => new SemaphoreSlim(1, 1));
        await sem.WaitAsync(ct);
        try
        {
            // 再次检查（sem 等待期间可能已被其他请求挂载）
            if (_disks.ContainsKey(config.DriveLetter))
                throw new DriveInUseException(config.DriveLetter);

            // 顺序构建：Pool → FS → StartDispatcher；失败逆序 Dispose
            PagePool? pool = null;
            VMemFileSystem? fs = null;
            try
            {
                pool = new PagePool(config.CapacityBytes, config.PageSizeKB * 1024,
                    preAllocate: config.PreAllocate);

                fs = new VMemFileSystem(pool, config);

                // StartDispatcher 在独立线程运行（它会阻塞）
                _ = Task.Run(() => fs.Host.Mount(config.DriveLetter), ct);

                _disks[config.DriveLetter] = new MountedDisk(config, pool, fs);
                _logger.Information("Mounted {Drive}: {Label} ({SizeGB}GB, page={PageKB}KB)",
                    config.DriveLetter, config.Label, config.CapacityBytes / 1073741824.0,
                    config.PageSizeKB);
            }
            catch (Exception ex)
            {
                _logger.Error(ex, "Mount {Drive} failed, cleaning up", config.DriveLetter);
                fs?.Dispose();    // 逆序 Dispose
                pool?.Dispose();
                throw;
            }
        }
        finally { sem.Release(); }
    }

    public async Task UnmountAsync(string driveLetter, CancellationToken ct)
    {
        if (!_disks.TryRemove(driveLetter, out var disk)) return;
        // drain timeout 5s（R14-4）；超时 ForceUnmount + ERR 日志
        using var drainCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        drainCts.CancelAfter(5000);
        await disk.StopAsync(drainCts.Token);
        disk.Dispose();
    }
}

public sealed class MountedDisk : IDisposable
{
    public DiskConfig Config { get; }
    public PagePool Pool { get; }
    public VMemFileSystem FileSystem { get; }
    public SnapshotManager? Snapshot { get; }

    // Oracle 审查：Dispose 严格顺序，防止 UAF
    public void Dispose()
    {
        try { FileSystem.Host.Unmount(); }       // StopDispatcher → 等待回调完成
        catch (Exception ex) { _logger.Error(ex, "Unmount error"); }

        try { Snapshot?.Dispose(); }
        catch (Exception ex) { _logger.Error(ex, "Snapshot dispose error"); }

        try { FileSystem.Dispose(); }            // 释放文件树（归还页面到 Pool）
        catch (Exception ex) { _logger.Error(ex, "FS dispose error"); }

        try { Pool.Dispose(); }                  // 释放物理内存
        catch (Exception ex) { _logger.Error(ex, "Pool dispose error"); }
    }

    public async Task StopAsync(CancellationToken ct)
    {
        // 如果启用快照 → 执行最终快照
        if (Snapshot != null)
        {
            try { await Snapshot.SnapshotAsync(ct); }
            catch (OperationCanceledException) { _logger.Warning("Final snapshot timed out"); }
            catch (Exception ex) { _logger.Error(ex, "Final snapshot failed"); }
        }
    }
}
```

### 3.1 Service 启动流程

```csharp
public class VMemWorker : BackgroundService
{
    private readonly RamDiskManager _manager;
    private readonly ConfigStore _config;
    private readonly PipeServer _pipeServer;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _logger.Information("Service starting, PID={Pid}, version={Version}",
            Environment.ProcessId, typeof(VMemWorker).Assembly.GetName().Version);

        // Step 1: 加载配置
        var cfg = _config.Load();
        _logger.Information("Config loaded: {DiskCount} auto-mount disks", cfg.AutoMountDisks.Count);

        // Step 2: 恢复自动挂载列表（按配置顺序逐个挂载）
        foreach (var disk in cfg.AutoMountDisks)
        {
            try
            {
                // 如果启用快照且镜像文件存在 → 先恢复再挂载
                if (disk.EnableSnapshot && File.Exists(disk.SnapshotImagePath))
                {
                    _logger.Information("Restoring {Drive} from {Path}",
                        disk.DriveLetter, disk.SnapshotImagePath);
                    await _manager.MountAndRestoreAsync(disk, ct);
                }
                else
                {
                    await _manager.MountAsync(disk, ct);
                }
                _logger.Information("Auto-mount {Drive} OK: {Label} ({SizeGB}GB)",
                    disk.DriveLetter, disk.Label, disk.CapacityBytes / 1073741824.0);
            }
            catch (Exception ex)
            {
                _logger.Error(ex, "Auto-mount {Drive} FAILED", disk.DriveLetter);
                // 单个磁盘失败不影响其他磁盘 → 继续下一个
            }
        }

        // Step 3: 启动 IPC 服务器（辩论裁决 R10：IPC Listen 不晚于 auto-mount）
        // IPC 与 auto-mount 并行启动，GUI 可在挂载期间连接监控进度
        _ = Task.Run(() => _pipeServer.ListenAsync(ct), ct);

        // 或：先 Listen 再 auto-mount（推荐）
        await _pipeServer.ListenAsync(ct);
    }
}
```

### 3.2 Service 停止与关机序列（辩论裁决 R14-5：普通串行 / PRESHUTDOWN 并行）

```
停止信号 (SERVICE_CONTROL_STOP 或 SERVICE_CONTROL_PRESHUTDOWN)
    │
    ├─ Step 1: 停止接受新的 IPC 请求
    │
    ├─ Step 2: 卸载策略分支
    │   ├─ STOP（普通）: 串行逐盘卸载
    │   └─ PRESHUTDOWN:  并行卸载，SemaphoreSlim(4) 限制并发
    │
    ├─ Step 3: 对每个已挂载磁盘
    │   ├─ 3a: 如果启用快照 → SnapshotManager.SnapshotAsync() 执行最终快照
    │   ├─ 3b: FspFileSystemStopDispatcher() → 停止处理新的 FS 请求
    │   ├─ 3c: 等待进行中的 FS 操作完成（drain timeout = 5s, R14-4）
    │   ├─ 3d: FspFileSystemDelete() → 卸载文件系统
    │   └─ 2e: PagePool.Dispose() → 释放所有 NativeMemory
    │
    ├─ Step 3: 关闭 IPC 管道
    │
    └─ Step 4: 记录关闭日志，退出
```

```csharp
public async Task StopAsync(CancellationToken ct)
{
    _logger.Information("Service stopping: {MountedCount} disks mounted", _manager.MountedDisks.Count);

    _pipeServer.StopAccepting();

    foreach (var disk in _manager.MountedDisks)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            if (disk.SnapshotEnabled)
            {
                await disk.Snapshot.SnapshotAsync(ct);
                _logger.Information("Final snapshot {Drive} OK in {Ms}ms",
                    disk.DriveLetter, sw.Elapsed.TotalMilliseconds);
            }
            await _manager.UnmountAsync(disk.DriveLetter, ct);
            _logger.Information("Unmounted {Drive}", disk.DriveLetter);
        }
        catch (OperationCanceledException)
        {
            _logger.Warning("Shutdown timeout, {Drive} may lose data", disk.DriveLetter);
        }
    }

    _logger.Information("Service stopped cleanly");
}
```

**PRESHUTDOWN 专用处理**：
- 注册时声明 `SERVICE_ACCEPT_PRESHUTDOWN`
- 默认 preshutdown timeout = 180 秒（Windows 默认值）
- Service 在 `sc create` 时声明 `SERVICE_CONTROL_PRESHUTDOWN_INFO` 为 180000ms
- 如果所有快照在 timeout 内完成 → 正常卸载；超时 → Windows 强杀进程 → 数据丢失（符合易失性语义）

### 3.3 ConfigStore - 完整配置体系

**配置文件位置**：`%ProgramData%\VMem\config.json`

```csharp
public sealed class ConfigStore
{
    private readonly string _configPath;      // %ProgramData%\VMem\config.json
    private readonly object _fileLock = new();
    private readonly ILogger _logger;

    public VMemConfig Load();
    public void Save(VMemConfig config);

    // 便捷方法
    public void AddAutoMountDisk(DiskConfig disk);
    public void RemoveAutoMountDisk(string driveLetter);
    public void UpdateAutoMountDisk(string driveLetter, Action<DiskConfig> modifier);
    public void UpdateGlobalSettings(Action<GlobalSettings> modifier);
}
```

**配置 JSON Schema（辩论裁决 R2/R10：V1 极简，V2 扩展）**：

```jsonc
// %ProgramData%\VMem\config.json
{
  "schemaVersion": 1,    // 必须有，V2 迁移用 IConfigMigrator

  // V1 全局设置（仅 logLevel，其余硬编码）
  "global": {
    "logLevel": "Warning"              // V1 默认 Warning（辩论裁决 R2）
    // V2+: memoryWarningPercent, enableReplayLog, logRetentionDays, preshutdownTimeoutMs
  },

  // 自动挂载磁盘列表
  // V1 config.json 仅写入 3 字段（辩论裁决 R2）
  // 其余字段代码硬编码：PageSize=4KB, EnableKernelCache=false, PreAllocate=false
  "autoMountDisks": [
    {
      "driveLetter": "R:",
      "capacityBytes": 2147483648,
      "label": "Browser Cache"
      // V2+: pageSizeKB, enableKernelCache, preAllocate, rootSddl, snapshot.*
    }
  ]
}
```

> **V1 对外契约 = 3 字段 JSON**；内部 DiskConfig record 结构可保留嵌套类型预留扩展，
> 但 **不持久化、不暴露 GUI** 扩展字段，等功能落地再扩 schema。

```csharp
public record VMemConfig(GlobalSettings Global, List<DiskConfig> AutoMountDisks);

public record GlobalSettings
{
    public int MemoryWarningPercent { get; init; } = 90;
    public string LogLevel { get; init; } = "Information";
    public bool EnableReplayLog { get; init; } = false;
    public int LogRetentionDays { get; init; } = 7;
    public int PreshutdownTimeoutMs { get; init; } = 180000;
}

/// DiskConfig 使用嵌套结构替代扁平的 11 参数构造函数，提升可读性和可维护性
public record DiskConfig
{
    public required string DriveLetter { get; init; }    // "R:"
    public required long CapacityBytes { get; init; }
    public string Label { get; init; } = "VMem RAM Disk";
    public string? CreatorSid { get; init; }             // 创建者 SID（多用户鉴权）

    public DiskPerformanceOptions Performance { get; init; } = new();
    public DiskSecurityOptions Security { get; init; } = new();
    public DiskSnapshotOptions Snapshot { get; init; } = new();
}

public record DiskPerformanceOptions
{
    public int PageSizeKB { get; init; } = 4;
    public bool EnableKernelCache { get; init; } = true;
    public uint FileInfoTimeoutMs { get; init; } = uint.MaxValue;
    public bool PreAllocate { get; init; } = false;
}

public record DiskSecurityOptions
{
    public string? RootSddl { get; init; }               // null = 按场景预设自动选择
    public bool AllowAllUsers { get; init; } = false;     // true → Everyone:FA
}

public record DiskSnapshotOptions
{
    public bool Enabled { get; init; } = false;
    public string? ImagePath { get; init; }               // null → 自动生成路径
    public int IntervalMinutes { get; init; } = 5;
}
```

**配置文件的读写规则**：

| 规则 | 说明 |
|------|------|
| **单写入者** | 只有 `VMem.Service` 通过 `ConfigStore` 写入，GUI 通过 IPC 间接修改 |
| **线程安全** | `ConfigStore` 内部持有 `SemaphoreSlim(1,1)`，所有读写方法异步串行化 |
| **原子写入** | `Save()` 先写 `config.json.tmp` → `File.Replace` 原子切换 → 防止写到一半断电损坏 |
| **自动备份** | 每次 `Save()` 成功后保留上一版为 `config.json.bak`（`File.Replace` 的第三个参数） |
| **损坏恢复** | `Load()` 时如果 `config.json` 解析失败，自动回退到 `config.json.bak` |
| **双重损坏** | `.json` 和 `.bak` 都损坏 → 使用内置默认配置（空磁盘列表）+ 记录 `FTL` 日志 |
| **升级兼容** | 读取时对缺失字段使用 `init` 默认值（新版本增加字段不影响旧配置读取） |

```csharp
public sealed class ConfigStore
{
    private readonly SemaphoreSlim _lock = new(1, 1);

    public async Task<VMemConfig> LoadAsync()
    {
        await _lock.WaitAsync();
        try
        {
            // 1. 尝试读取 config.json
            // 2. 解析失败 → 尝试 config.json.bak
            // 3. 都失败 → 返回默认配置 + 记录 FTL 日志
        }
        finally { _lock.Release(); }
    }

    public async Task SaveAsync(VMemConfig config)
    {
        await _lock.WaitAsync();
        try
        {
            var tmpPath = _configPath + ".tmp";
            await File.WriteAllTextAsync(tmpPath, JsonSerializer.Serialize(config, _jsonOptions));
            File.Replace(tmpPath, _configPath, _configPath + ".bak");
            // File.Replace 是 NTFS 原子操作：成功 → .json 是新内容，.bak 是旧内容
            // 失败/断电 → .json 仍是旧内容（安全）
        }
        finally { _lock.Release(); }
    }
}
```

### 3.4 GUI 本地配置（独立于 Service 配置）

GUI 应用有自己的本地配置，保存纯客户端侧的偏好，不通过 IPC 传递：

**配置文件位置**：`%LocalAppData%\VMem\gui-settings.json`

```jsonc
{
  "theme": "System",              // "Light" | "Dark" | "System"
  "minimizeToTrayOnClose": true,  // 关闭窗口时最小化到托盘
  "launchAtLogin": true,          // 用户登录时自动启动 GUI
  "logLevel": "Information",      // GUI 日志级别
  "windowPosition": { "x": 100, "y": 100, "width": 1024, "height": 680 }
}
```

**谁管理什么配置**：

| 配置项 | 存储位置 | 管理方 | 修改入口 |
|--------|---------|--------|---------|
| 自动挂载列表 | `%ProgramData%\VMem\config.json` | Service | GUI 设置页 → IPC |
| 磁盘参数（容量/页大小/缓存） | 同上 | Service | GUI 创建对话框 → IPC |
| 快照开关/周期 | 同上 | Service | GUI 磁盘设置对话框 → IPC |
| 内存告警阈值 | 同上 | Service | GUI 设置页 → IPC |
| 日志级别 (Service) | 同上 | Service | GUI 设置页 → IPC |
| GUI 主题 | `%LocalAppData%\VMem\gui-settings.json` | GUI | GUI 设置页 → 本地保存 |
| GUI 最小化到托盘 | 同上 | GUI | GUI 设置页 → 本地保存 |
| GUI 随登录启动 | 同上 + 注册表 HKCU\...\Run | GUI | GUI 设置页 → 注册表 |
| GUI 窗口位置 | 同上 | GUI | 自动保存 |

---

## 4. 可靠性设计

### 4.1 Service 崩溃自动恢复

安装包注册时声明故障恢复策略（已在子方案 ⑤ 安装流程中配置）：

```
sc failure VMem reset=86400 actions=restart/60000/restart/60000/restart/60000
```

| 崩溃次数 | 行为 | 说明 |
|---------|------|------|
| 第 1 次 | 60 秒后重启 | 给系统一些缓冲时间 |
| 第 2 次 | 60 秒后重启 | 再试一次 |
| 第 3 次 | 60 秒后重启 | 最后尝试 |
| 后续 | 不再重启 | 避免无限崩溃循环 |
| 24 小时后 | 重置崩溃计数 | `reset=86400` |

Service 重启后走正常启动流程（读 config → 自动挂载列表 → 恢复快照），效果等同于重启电脑。

### 4.2 系统 OOM（内存耗尽）

Windows 不像 Linux 有 OOM Killer，但低内存时行为复杂：

| 场景 | 表现 | VMem 应对 |
|------|------|----------|
| 物理内存不足 | `NativeMemory.AllocZeroed` 仍能成功（Windows 会将其他进程页面换出到 pagefile） | 无需特殊处理，但性能急剧下降 |
| 提交限制(Commit limit)耗尽 | `NativeMemory.AllocZeroed` 返回 null / 抛 OutOfMemoryException | `TryRent` 捕获并返回 `false` → 映射为 `STATUS_DISK_FULL` |
| 系统极度缺页 | Service 进程被系统换出，响应极慢 | 健康检查检测到响应延迟 > 5s → `WRN` 日志 |

**主动内存保护**：
- `HealthChecker` 每 30 秒检查系统可用物理内存
- 低于 `global.memoryWarningPercent` 阈值 → 推送 `MemoryWarning` 事件到 GUI
- GUI 弹出 Snackbar 警告："系统内存不足，请注意保存数据或卸载部分 RAM 盘"
- 日志记录当前各磁盘内存占用，供 AI 分析是否需要建议用户缩减容量

### 4.3 快照写入磁盘满

| 场景 | 处理 |
|------|------|
| 写入 `.vmem.tmp` 时磁盘空间不足 | 捕获 `IOException` → 删除 `.vmem.tmp` → 保留上一次 `.vmem` 不受影响 → `ERR` 日志 |
| 持续磁盘满导致多次快照失败 | 推送 `SnapshotFailed` 事件到 GUI，Snackbar 警告 |
| `.vmem` 文件被其他程序锁定 | 捕获 `IOException` → 跳过本次快照 → `WRN` 日志 |

---

## 5. 磁盘全生命周期流程

### 5.1 创建磁盘（含自动挂载）

```
用户点击 GUI [创建] 按钮
    │
    ├─ GUI 构建 CreateDiskRequest（含 AutoMount=true/false）
    ├─ → IPC → Service
    │
    └─ Service 处理：
        ├─ Step 1: 验证盘符未占用
        ├─ Step 2: 验证系统可用内存足够
        ├─ Step 3: RamDiskManager.MountAsync(diskConfig)
        │   ├─ new PagePool(capacity, pageSize)
        │   ├─ new VMemFileSystem(pool, ...)
        │   └─ FspFileSystemStartDispatcher() → 磁盘可用
        │
        ├─ Step 4: 如果 AutoMount == true
        │   ├─ ConfigStore.AddAutoMountDisk(diskConfig)
        │   └─ config.json 已更新 → 下次 Service 启动自动恢复此磁盘
        │
        ├─ Step 5: 推送 DiskMounted 事件 → GUI 更新仪表盘
        └─ Step 6: 返回 IpcResponse(Success=true)
```

### 5.2 卸载磁盘

```
用户点击磁盘卡片 [卸载] 按钮
    │
    ├─ GUI 弹出确认对话框：
    │   "确定卸载 R: (Browser Cache)？磁盘上的数据将丢失。"
    │   [ ] 同时从自动挂载列表移除
    │   [取消] [卸载]
    │
    ├─ → IPC → RemoveDiskRequest(DriveLetter="R:", RemoveFromAutoMount=checked)
    │
    └─ Service 处理：
        ├─ Step 1: 如果启用快照 → 执行最终快照
        ├─ Step 2: FspFileSystemStopDispatcher()
        ├─ Step 3: 等待进行中操作完成 (drain 5s)
        ├─ Step 4: FspFileSystemDelete() → 磁盘从系统消失
        ├─ Step 5: PagePool.Dispose() → 释放内存
        ├─ Step 6: 如果 RemoveFromAutoMount == true
        │   └─ ConfigStore.RemoveAutoMountDisk("R:")
        ├─ Step 7: 推送 DiskUnmounted 事件
        └─ Step 8: 返回 IpcResponse(Success=true)
```

### 5.3 编辑磁盘配置

```
用户点击磁盘卡片 [设置] 按钮
    │
    ├─ GUI 弹出磁盘设置对话框（显示当前配置）
    │   可热修改：卷标、自动挂载开关、快照开关/周期
    │   需要重新挂载：容量、页大小、内核缓存、SDDL
    │
    ├─ → IPC → UpdateDiskConfigRequest(DriveLetter, 变更字段)
    │
    └─ Service 处理：
        ├─ 热修改项 → 直接生效 + 更新 ConfigStore
        └─ 冷修改项 → 返回提示"需要卸载后重新创建"
```

### 5.4 开机自动恢复

```
Windows 启动 → SCM 启动 VMem.Service
    │
    └─ VMemWorker.ExecuteAsync()
        ├─ 读取 config.json → autoMountDisks[]
        │
        └─ 对列表中每个磁盘：
            ├─ 检查盘符是否被占用（已有物理磁盘/光驱）
            │   └─ 已占用 → 跳过 + 记录 ERR 日志
            │
            ├─ 如果 EnableSnapshot && 镜像文件存在
            │   ├─ RestoreAsync() → 从 .vmem 恢复内容
            │   └─ MountAsync() → 磁盘可用（内容已恢复）
            │
            └─ 否则
                └─ MountAsync() → 空磁盘可用
```

### 5.5 关机序列

```
Windows 关机
    │
    ├─ SCM 发送 SERVICE_CONTROL_PRESHUTDOWN
    │   ├─ 对每个启用快照的磁盘 → SnapshotAsync() 最终快照
    │   ├─ 对每个磁盘 → UnmountAsync() 优雅卸载
    │   └─ 完成 → Service 退出
    │
    ├─ 超时 (180s) → Windows 强杀进程
    │   └─ 未快照完成的磁盘 → 下次启动时使用上一次成功的快照恢复（或空盘）
    │
    └─ GUI (VMem.App) 进程
        ├─ 收到 WM_QUERYENDSESSION → 保存窗口位置等本地配置
        └─ 正常退出（GUI 不管磁盘卸载，全由 Service 负责）
```

---

## 6. CLI 工具（Phase 1）

Phase 1 先实现简单 CLI 直接挂载（不经过 Service），用于验证核心逻辑：

```
vmem mount R: --size 1G [--page-size 4] [--label "RAM Disk"] [--kernel-cache]
vmem unmount R:
vmem status
```

Phase 3 CLI 增加通过 IPC 控制 Service 的命令：

```
vmem service install
vmem service start/stop
vmem create R: --size 1G    # 通过 Named Pipe 发送 CreateDiskRequest
vmem remove R:
```

---

## 7. 日志与可观测性（L3 范式补丁）

> 详见 [子方案 ⑦ - 日志与可观测性](./vmem_07_logging_observability.plan.md)

### 7.1 IPC 请求追踪

每个 IPC 请求携带 `RequestId`（已在 §2.2 的 `IpcMessage` 基类中定义），Service 端将其映射为 `CorrelationId`，实现 GUI↔Service 跨进程追踪：

```csharp
// GUI 端发送
var request = new CreateDiskRequest("R:", ...) { RequestId = Guid.NewGuid().ToString("N")[..8] };

// Service 端接收
OperationContext.CorrelationId = request.RequestId;
_logger.Information("IPC received {Type} requestId={RequestId} driveLetter={DriveLetter}",
    request.Type, request.RequestId, request.DriveLetter);
```

### 7.2 IPC 通信日志

| 事件 | 日志级别 | 记录内容 |
|------|---------|---------|
| 请求到达 | `INF` | requestId, Type, 关键参数摘要 |
| 请求处理完成 | `INF` | requestId, Type, 耗时, 成功/失败 |
| 请求处理失败 | `ERR` | requestId, Type, 异常类型, 异常消息, 堆栈 |
| 客户端连接 | `INF` | 客户端 PID（通过 `GetNamedPipeClientProcessId`） |
| 客户端断开 | `INF` | 客户端 PID, 连接持续时间 |
| 事件推送发送 | `DBG` | 事件类型, 目标客户端数 |
| IPC 往返 > 500ms | `WRN` | requestId, Type, 耗时, 瓶颈分析 |

### 7.3 运行时日志级别切换

通过 §2.2 中的 `UpdateGlobalSettingsRequest(LogLevel=...)` IPC 命令实现。Service 端使用 Serilog `LoggingLevelSwitch` 热切换，无需重启：

```csharp
private readonly LoggingLevelSwitch _levelSwitch = new(LogEventLevel.Information);

// 处理 UpdateGlobalSettings 请求
if (request.LogLevel is not null)
{
    _levelSwitch.MinimumLevel = Enum.Parse<LogEventLevel>(request.LogLevel);
    _logger.Information("Log level changed to {Level} by IPC request {RequestId}",
        request.LogLevel, request.RequestId);
}
```

CLI 同样支持：`vmem log-level Debug` / `vmem log-level Information`

### 7.4 Service 生命周期日志

| 事件 | 日志级别 | 记录内容 |
|------|---------|---------|
| Service 启动 | `INF` | 版本号, PID, 运行身份, .NET 版本, WinFsp 版本 |
| 自动挂载恢复 | `INF` | 每个盘：盘符, 容量, 页大小, 是否成功 |
| 自动挂载失败 | `ERR` | 盘符, 失败原因, 配置详情 |
| Service 停止 | `INF` | 正常/异常, 已挂载盘数, 运行时长 |
| PRESHUTDOWN 收到 | `INF` | 剩余时间, 需快照的盘数 |
| 未处理异常 | `FTL` | 异常类型, 消息, 完整堆栈 |

### 7.5 配置变更审计

`ConfigStore` 每次保存时记录配置 diff，便于 AI 追踪配置变化：

```csharp
public void SaveAutoMountList(List<DiskConfig> newDisks)
{
    var oldDisks = LoadAutoMountList();
    var diff = ComputeDiff(oldDisks, newDisks);
    _logger.Information("Config updated: {Added} added, {Removed} removed, {Modified} modified. " +
        "Details: {@Diff}", diff.Added, diff.Removed, diff.Modified, diff);
    // ... 实际保存 ...
}
```

审计日志示例：
```json
{
  "level": "INF",
  "msg": "Config updated",
  "props": {
    "added": ["T:"],
    "removed": [],
    "modified": [{"drive": "R:", "field": "CapacityBytes", "old": 1073741824, "new": 2147483648}]
  }
}
```
