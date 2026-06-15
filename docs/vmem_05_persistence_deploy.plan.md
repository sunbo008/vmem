---
name: "VMem 子方案 ⑤ - 持久化与部署"
overview: "脏页增量快照、.vmem 镜像格式、关机处理、Native AOT 编译、Inno Setup 安装包。"
todos:
  - id: snapshot-manager
    content: "实现 SnapshotManager：脏页追踪 + 定时快照 + 双缓冲原子写入"
    status: pending
  - id: image-format
    content: "实现 .vmem 镜像格式：Header + PageBitmap + PageData + TreeData"
    status: pending
  - id: restore
    content: "实现启动时镜像恢复（RestoreAsync）"
    status: pending
  - id: preshutdown
    content: "实现 SERVICE_CONTROL_PRESHUTDOWN 最终快照"
    status: pending
  - id: native-aot
    content: "CLI 工具 Native AOT 发布：System.Text.Json SourceGen + AOT 兼容验证"
    status: pending
  - id: installer
    content: "Inno Setup 安装包：捆绑 WinFsp + 注册 Service + 升级保留配置"
    status: pending
isProject: false
---

# 子方案 ⑤ - 持久化与部署

> 父方案：[VMem RAM Disk Design](./vmem_ram_disk_design_031231c8.plan.md)
> 对应代码：`src/VMem.Core/Snapshot/` + `installer/`
> Phase：4

---

## 1. 持久化设计理念

RAM 盘本质是**易失性存储**。持久化作为 **best-effort** 可选功能：
- **不保证**关机/崩溃/断电下的数据完整性
- 使用**增量脏页快照**替代不可靠的关机全量写入

---

## 2. 周期性脏页快照

### 2.1 SnapshotManager（辩论裁决 R3：V1 骨架 + 无周期）

> **V1 范围**：仅 preshutdown + 显式卸载触发，**无 Timer 周期快照**。默认关闭。

```csharp
public sealed class SnapshotManager : IDisposable
{
    private readonly string _imagePath;
    private readonly PagePool _pool;
    private readonly DirectoryTree _tree;
    // V1 无 Timer（周期快照推迟到 Phase 4+）

    // 脏页追踪（辩论裁决 R3：ulong[] + Interlocked.Or + 双缓冲 swap）
    private ulong[] _dirtyActive;    // Write 路径：Interlocked.Or 标记
    private ulong[] _dirtyStaging;   // Snapshot 消费

    /// Write Phase 3 末尾调用，热路径无锁
    public void MarkDirty(int pageIndex)
    {
        int chunk = pageIndex >> 6;
        ulong mask = 1UL << (pageIndex & 63);
        Interlocked.Or(ref _dirtyActive[chunk], mask);
    }

    /// V1 仅两条路径调用：
    /// 1) SERVICE_CONTROL_PRESHUTDOWN / StopAsync
    /// 2) 用户 IPC Unmount(drive, finalSnapshot: true)
    public Task SnapshotAsync(SnapshotTrigger trigger, CancellationToken ct);

    public Task RestoreAsync(CancellationToken ct);
}

public enum SnapshotTrigger { Shutdown, ManualUnmount }
```

**V1 快照流程（两阶段模型）**：

```
Phase 0 — 一致性点（持 freeze，P99 ≤ 50ms）
  ├─ FspFileSystemStopDispatcher()          // 拒新请求
  ├─ await DrainInflightOps(timeout)        // preshutdown: 5000ms
  ├─ dirtySnapshot = AtomicSwapDirtyBitmap()// 双缓冲指针交换
  ├─ treeJson = DirectoryTree.Serialize()   // 同一时间点
  ├─ pageRefs = PagePool.PinPages(dirtySnapshot)
  └─ FspFileSystemStartDispatcher()         // 立即恢复 IO（preshutdown 不恢复）

Phase 1 — 异步 IO（无 freeze，不阻塞 FS）
  ├─ 写 .vmem.tmp（Header + PageBitmap + PageData + TreeData）
  └─ File.Replace → 原子切换
```

```csharp
DirtyPageSet CaptureAndResetDirtyPages()
{
    // 指针交换（freeze 内，单线程安全）
    (_dirtyActive, _dirtyStaging) = (_dirtyStaging, _dirtyActive);
    Array.Clear(_dirtyActive, 0, _dirtyActive.Length);
    return new DirtyPageSet(_dirtyStaging);
}
```

**配置参数（V1 默认关闭）**：
```jsonc
{
  "EnableSnapshot": false,
  "SnapshotImagePath": "C:\\ProgramData\\VMem\\snapshots\\R.vmem"
  // V2+: SnapshotIntervalMinutes (1-60)
}
```

### 2.2 镜像文件格式（.vmem）

```
Header (固定 4KB):
  Magic:        "VMEM" (4 bytes)
  Version:      uint32
  PageSize:     uint32
  TotalPages:   uint64
  UsedPages:    uint64
  TreeOffset:   uint64  (目录树序列化数据的偏移)
  TreeSize:     uint64
  Checksum:     SHA256 (32 bytes)

Body:
  PageBitmap:   ceil(TotalPages/8) bytes  (1=已分配, 0=稀疏)
  PageData:     连续存储所有 bit=1 的页数据
  TreeData:     JSON 序列化（文件名、大小、时间戳、SD、页索引映射）
```

### 2.3 关机处理（详见[子方案 ③ §3.2](./vmem_03_ipc_service.plan.md)）

完整关机序列由 Service 统一管理：

```
SERVICE_CONTROL_PRESHUTDOWN 到达
    │
    ├─ Step 1: 停止接受新 IPC 请求
    │
    ├─ Step 2: 对每个启用快照的磁盘
    │   └─ SnapshotManager.SnapshotAsync() → 最终增量快照
    │
    ├─ Step 3: 对每个磁盘执行优雅卸载
    │   ├─ FspFileSystemStopDispatcher()
    │   ├─ drain 进行中操作 (5s timeout)
    │   ├─ FspFileSystemDelete()
    │   └─ PagePool.Dispose()
    │
    └─ Step 4: Service 退出
```

**关键参数**：
- Service 注册时声明 `SERVICE_ACCEPT_PRESHUTDOWN`
- Preshutdown timeout = 180 秒（在 `sc create` 时通过 `SERVICE_PRESHUTDOWN_INFO` 配置）
- 可通过 `config.json` 的 `global.preshutdownTimeoutMs` 调整
- 超时未完成 → Windows 强杀进程 → 未快照的数据丢失（符合易失性语义）
- 下次启动恢复时使用**上一次成功的快照**（不是最终快照）

---

## 3. Native AOT（CLI 工具）

hooyao/RamDrive 已验证 WinFsp.Native + Native AOT 的可行性：

| 指标 | 框架依赖发布 | Native AOT |
|------|-------------|-----------|
| 文件大小 | ~80MB (.NET Runtime) | ~10-15MB 单文件 |
| 启动时间 | ~500ms | < 100ms |
| JIT 延迟 | 首次回调有冷启动 | 零 |
| 运行时依赖 | .NET 9 Runtime | 无 |

**VMem AOT 演进路线（辩论裁决 R18）**：

| Phase | 项目 | 发布方式 | 说明 |
|-------|------|---------|------|
| P1 | VMem.Core + VMem.Cli | **AOT CI 门禁**（build only） | 保证代码 AOT 兼容 |
| P3 | VMem.Service | 框架依赖 + ReadyToRun | 启动快 + 无需独立 Runtime |
| P4 | VMem.Cli + VMem.Service | **Native AOT 单文件** | 零依赖部署 |
| 全期 | VMem.App (WPF) | 框架依赖，**不 Trim 不 AOT** | WPF 依赖反射 |

**AOT 约束**：
- `VMem.Core` 标记 `IsAotCompatible = true`
- WinFsp 绑定使用 `WinFsp.Native`（非反射）
- `System.Text.Json` 必须使用 **Source Generator**（`IpcJsonContext`）
- `Serilog` 配置禁止使用反射 Sink（`ReadFrom.Configuration` → 手动 `WriteTo`）
- CI 门禁：PR=`dotnet build /p:PublishAot=true`，main=`dotnet publish -r win-x64 /p:PublishAot=true`

---

## 4. Inno Setup 安装包

### 安装流程

```
1. 检测 WinFsp 是否已安装
   ├─ 已安装 → 跳过
   └─ 未安装 → 解压并静默安装捆绑的 WinFsp MSI

2. 创建目录结构
   ├─ %ProgramFiles%\VMem\              → 程序文件
   ├─ %ProgramData%\VMem\               → Service 配置
   ├─ %ProgramData%\VMem\logs\          → 日志
   └─ %ProgramData%\VMem\snapshots\     → 快照镜像

3. 复制 VMem.Service.exe, VMem.App.exe, VMem.Cli.exe 到 Program Files

4. 注册 Windows Service
   sc create VMem binPath="..." start=auto obj=LocalSystem
   sc preshutdown VMem 180000
   sc description VMem "VMem RAM Disk Service"
   sc failure VMem reset=86400 actions=restart/60000/restart/60000/restart/60000

5. 初始化默认配置
   → 写入 %ProgramData%\VMem\config.json（默认空自动挂载列表）

6. 启动 Service
   sc start VMem

7. 创建开始菜单快捷方式 → VMem.App.exe

8. 可选：注册 GUI 开机自启
   → 写入 HKCU\Software\Microsoft\Windows\CurrentVersion\Run
```

### 升级行为（辩论裁决 R17）

```
1. sc stop VMem                             → 停止 Service（触发快照+卸载序列）
2. 等待 Service 完全停止 (timeout 60s)
3. 保留 %ProgramData%\VMem\config.json      → 配置不丢失
4. 保留 %ProgramData%\VMem\snapshots\*.vmem → 快照不丢失
5. 保留 %ProgramData%\VMem\logs\            → 日志不丢失
6. 替换 %ProgramFiles%\VMem\ 下的程序文件
7. sc start VMem                             → 启动新版 Service（自动恢复磁盘）
```

**配置迁移**（R17-3）：
- `config.json` 包含 `schemaVersion` 字段
- `ConfigStore.Load()` 检测版本差异 → 执行 `IConfigMigrator` 链 → 就地 Save 写回
- 迁移日志记录旧/新 schema 版本

**ACL 权限**（R17-4）：

| 路径 | 权限 |
|------|------|
| `%ProgramFiles%\VMem\` | 标准 ProgramFiles ACL（Admin 写、Users 读执行） |
| `%ProgramData%\VMem\` | SYSTEM + Administrators 完全控制；Users 读+执行 |
| `config.json` | 继承 ProgramData ACL |
| `logs\` | 继承 ProgramData ACL |

**WinFsp 检测**（R17-1）：安装器启动时检测 WinFsp → 未安装则从嵌入资源解压 MSI 静默安装；禁止纯拒绝或在线下载。

### 卸载

```
1. sc stop VMem                         → 停止 Service（触发快照+卸载）
2. 等待 Service 完全停止
3. sc delete VMem                       → 删除 Service 注册
4. 删除 HKCU\...\Run 中的 GUI 自启注册
5. 删除 %ProgramFiles%\VMem\           → 程序文件
6. 询问用户是否删除数据：
   ├─ 是 → 删除 %ProgramData%\VMem\    （配置+快照+日志全部清除）
   └─ 否 → 保留（用户可能要重装或手动查看日志）
7. 不卸载 WinFsp（可能被其他程序使用）
```

---

## 5. 日志与可观测性（L3 范式补丁）

> 详见 [子方案 ⑦ - 日志与可观测性](./vmem_07_logging_observability.plan.md)

### 5.1 快照过程全程日志

```csharp
public async Task SnapshotAsync(CancellationToken ct)
{
    var sw = Stopwatch.StartNew();
    var dirtyCount = _dirtyPages.CountSet();

    _logger.Information("Snapshot started: drive={Drive} dirtyPages={DirtyPages} " +
        "estimatedBytes={EstBytes}",
        _driveLetter, dirtyCount, dirtyCount * _pool.PageSize);

    // ... 写入 .vmem.tmp ...

    _logger.Information("Snapshot page data written: {WrittenPages}/{TotalDirty} pages, " +
        "{WrittenMB}MB in {Ms}ms",
        writtenPages, dirtyCount, writtenMB, sw.Elapsed.TotalMilliseconds);

    // ... 目录树序列化 ...

    _logger.Information("Snapshot tree serialized: {FileCount} files, {TreeSizeKB}KB",
        fileCount, treeSizeKB);

    // ... File.Replace 原子切换 ...

    _logger.Information("Snapshot completed: drive={Drive} totalMs={Ms} " +
        "imageSize={ImageSizeMB}MB dirtyPages={DirtyPages}",
        _driveLetter, sw.Elapsed.TotalMilliseconds, imageSizeMB, dirtyCount);
}
```

| 事件 | 日志级别 | 记录内容 |
|------|---------|---------|
| 快照开始 | `INF` | 盘符, 脏页数, 预估字节数 |
| 页数据写入完成 | `INF` | 已写页数/总脏页数, MB 数, 耗时 |
| 目录树序列化完成 | `INF` | 文件数, 序列化大小 |
| 原子切换完成 | `INF` | 盘符, 总耗时, 镜像大小 |
| 快照失败 | `ERR` | 盘符, 异常类型, 消息, 已写入页数（部分进度） |
| 快照耗时 > 10s | `WRN` | 盘符, 耗时, 脏页数, 瓶颈分析（IO/序列化） |
| 脏页标记 | `VRB` | 页索引（仅 Verbose 模式下记录，用于回放） |

### 5.2 恢复过程逐步校验日志

```csharp
public async Task RestoreAsync(CancellationToken ct)
{
    var sw = Stopwatch.StartNew();

    // Step 1: Header 校验
    _logger.Information("Restore started: image={Path}", _imagePath);
    var header = ReadHeader();
    _logger.Information("Restore header OK: magic={Magic} version={Version} " +
        "pageSize={PageSize} totalPages={Total} usedPages={Used}",
        header.Magic, header.Version, header.PageSize, header.TotalPages, header.UsedPages);

    // Step 2: Checksum 验证
    if (!VerifyChecksum(header))
    {
        _logger.Error("Restore FAILED: checksum mismatch, image={Path}", _imagePath);
        return;
    }
    _logger.Information("Restore checksum verified");

    // Step 3: 页数据恢复
    _logger.Information("Restore loading {UsedPages} pages...", header.UsedPages);
    // ... 逐页恢复 ...
    _logger.Information("Restore pages loaded: {Loaded}/{Total} in {Ms}ms",
        loadedPages, header.UsedPages, sw.Elapsed.TotalMilliseconds);

    // Step 4: 目录树恢复
    _logger.Information("Restore loading directory tree ({TreeSizeKB}KB)...", header.TreeSize / 1024);
    // ... 反序列化 ...
    _logger.Information("Restore tree loaded: {FileCount} files, {DirCount} dirs",
        fileCount, dirCount);

    // Step 5: 完成
    _logger.Information("Restore completed: drive={Drive} totalMs={Ms} " +
        "files={Files} pages={Pages}",
        _driveLetter, sw.Elapsed.TotalMilliseconds, fileCount, loadedPages);
}
```

### 5.3 关机处理日志

```csharp
// PRESHUTDOWN 处理
_logger.Information("PRESHUTDOWN received: {MountedDisks} disks to snapshot, " +
    "timeout={TimeoutSec}s", mountedDisks.Count, timeoutSec);

foreach (var disk in mountedDisks)
{
    _logger.Information("PRESHUTDOWN snapshot {Drive} started", disk.DriveLetter);
    // ... 快照 ...
    _logger.Information("PRESHUTDOWN snapshot {Drive} completed in {Ms}ms",
        disk.DriveLetter, sw.Elapsed.TotalMilliseconds);
}

_logger.Information("PRESHUTDOWN all snapshots completed, total={Ms}ms", totalSw.Elapsed.TotalMilliseconds);
```
