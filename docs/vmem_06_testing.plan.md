---
name: "VMem 子方案 ⑥ - 测试策略"
overview: "分层测试体系（单元/集成/合规/压力/性能/泄漏）、CI 流水线、正确性不变量验证。"
todos:
  - id: unit-tests
    content: "编写单元测试：PagePool/PagedFileContent/DirectoryTree/SecurityDescriptor"
    status: pending
  - id: integration-tests
    content: "编写集成测试：挂载真实 WinFsp FS + 文件 CRUD/权限/并发"
    status: pending
  - id: winfsp-compliance
    content: "运行 winfsp-tests 官方合规测试套件"
    status: pending
  - id: cache-coherency
    content: "编写缓存一致性测试：EnableKernelCache=true 下 Rename→Read"
    status: pending
  - id: stress-tests
    content: "编写压力测试：8 线程并发 10 分钟无崩溃"
    status: pending
  - id: benchmarks
    content: "编写 BenchmarkDotNet 性能基准 + 与 hooyao/RamDrive 对比"
    status: pending
  - id: ci-pipeline
    content: "配置 GitHub Actions CI：Build → Test → Benchmark"
    status: pending
isProject: false
---

# 子方案 ⑥ - 测试策略

> 父方案：[VMem RAM Disk Design](./vmem_ram_disk_design_031231c8.plan.md)
> 对应代码：`tests/`
> Phase：全程（随各模块同步编写）

---

## 1. 分层测试体系（辩论裁决 R9：投入比 40% 单测 : 60% 合规+集成）

| 层级 | 框架 | 覆盖目标 | V1 数量 | CI 运行 |
|------|------|----------|--------|---------|
| 单元测试 | xUnit | PagePool(10) + PagedFileContent(8) + DirectoryTree(6) | **24 条** | 每次 push |
| 集成测试 | xUnit | CRUD、Rename、并发 8 线程、SD 继承、KernelCache Rename、DISK_FULL | **6 条** | 每次 push（需 WinFsp） |
| 合规测试 | winfsp-tests | 已支持特性 **100%**；不支持特性 **explicit skip** | 1 suite | main + nightly |
| 压力 smoke | 自定义 | 8 线程 120 秒无崩溃 + 泄漏检测 | **1 条** | 每次 push |
| 压力 full | 自定义 | 8 线程 600 秒 + 10 万文件循环 | **2 条** | nightly |
| 微基准 | BenchmarkDotNet | Rent/Return/Batch/CAS（无 WinFsp 依赖） | **4 条** | main artifact |
| IPC / GUI | — | V1 不测 | 0 | Phase 2-3 |

> **合计：约 32 条自动化用例 + 1 合规套件**（Phase 1 MVP）

### 新增交付物
- `docs/known-skips.md`：winfsp-tests skip 理由 + WinFsp 版本 + 计划 Phase。**禁止** open-ended known failures。

**预期 Phase 1 Skip 列表**（辩论裁决 R66-R70）：

| winfsp-tests 类别 | Skip 理由 | 计划 Phase |
|-------------------|-----------|-----------|
| `stream_*` | Named Streams 未实现 | Phase 2 考虑 |
| `reparse_*` | Reparse Points 未实现 | Phase 2 考虑 |
| `ea_*` | Extended Attributes 不实现 | ✗ |
| `oplock_*` | Oplock 不实现（RAM 盘无远程场景） | ✗ |
| `hardlink_*` | Hard Links 未实现 | Phase 2 考虑 |
| `querydir_single` | GetDirInfoByName 未实现（V1 用 ReadDirectory） | Phase 2 |

> 所有非 skip 测试 Phase 1 必须 **100% 通过**。

---

## 2. 单元测试详细覆盖

### PagePoolTests

| 测试 | 验证点 |
|------|--------|
| Rent_Return_RoundTrip | 租借后归还，页面回到 free stack |
| TryRent_WhenExhausted_ReturnsFalse | 达到上限后返回 false |
| TryRentBatch_ReturnsPartial | 批量请求部分满足 |
| Return_ClearsMemory | 归还后页面全零 |
| TryReserve_AtomicUnderContention | 8 线程并发 Reserve CAS 正确 |
| TryReserve_ExceedsCapacity_ReturnsFalse | 预留超限返回 false |
| TryRentFromReservation_DecrementsReserved | 预留转 Rent 计数正确 |
| Dispose_DetectsLeak | 有未归还页时 ERR 日志 + Debug.Fail |
| LazyAllocation_AllocsOnDemand | 首次 Rent 才触发 NativeMemory.Alloc |
| PreAllocate_AllocsAll | PreAllocate=true 构造时全部分配 |
| TryReserve_PostConditionRollback | CAS 成功但 post-check 超额时回退 |
| Return_AfterDispose_StillReturns | Dispose 后延迟 Return 不抛异常 |
| Return_NullPage_ThrowsArgException | Return(0) 抛 ArgumentException |
| BatchLease_Commit_AfterDispose_Throws | Dispose 后 Commit 抛 ObjectDisposedException |
| BatchLease_DoubleCommit_Throws | 同一页二次 Commit 抛 InvalidOperationException |

### PagedFileContentTests

| 测试 | 验证点 |
|------|--------|
| Write_Read_RoundTrip | 写入后读回一致 |
| Read_SparseRegion_ReturnsZeros | 未写入区域全零 |
| Write_CrossPageBoundary | 跨页写入正确拼接 |
| ThreePhaseWrite_UnderContention | 8 线程同时写不同偏移 |
| ThreePhaseWrite_TOCTOU_ReturnsSurplus | Phase 2~3 并发填充后归还多余 |
| SetLength_Extend_ReservesCapacity | 扩展预留配额 |
| SetLength_Truncate_FreesPages | 截断归还页面 + Unreserve |
| CapacityExhaustion_ReturnsError | 池耗尽返回 STATUS_DISK_FULL |
| SetLength_Truncate_ClearsPartialPage | 截断到页中间时尾部字节清零 |
| Write_ZeroLength_NoOp | 写入 0 字节返回 SUCCESS 无副作用 |
| Dispose_ReleasesAllPages | Dispose 后所有页归还到 Pool |
| Dispose_UnreservesRemaining | Dispose 后预留计数归零 |

### DirectoryTreeTests

| 测试 | 验证点 |
|------|--------|
| Lookup_CaseInsensitive | 路径大小写不敏感 |
| TryCreate_ConcurrentSameName | 并发创建同名只有一个成功 |
| Rename_CrossDirectory_NoDeadlock | 跨目录 Rename 不死锁 |
| Rename_CaseOnly_SameDirectory | `FOO→foo` 同目录大小写变更不冲突（R19） |
| Rename_ReplaceExisting | replaceIfExists 正确替换 |
| TryRemove_NonEmptyDir_Fails | 非空目录删除失败 |
| EnumerateDirectory_SnapshotIsolation | 枚举期间并发修改不影响结果 |
| Children_OrdinalIgnoreCase | ConcurrentDictionary 使用 OrdinalIgnoreCase（R20 P0） |

### 边界场景测试（辩论裁决 R19/R21-R30 新增）

| 测试 | 验证点 |
|------|--------|
| ZeroByteFile_ContentNotNull | 0 字节文件 Content != null（R19） |
| Read_BeyondFileSize_Clamped | Read 超过 FileSize 返回截断数据 |
| Read_SparseHole_ReturnsZeros | 稀疏区域读取全零填充（R11） |
| LongPathComponent_Rejected | 组件名 >255 字符被拒绝（R19） |
| GetVolumeInfo_FreeSize_Matches | FreeSize = Capacity - Used - Reserved（R22） |
| Overwrite_TruncatesAndResetsTimestamp | Overwrite 截断内容并重置时间戳（R21） |
| SetBasicInfo_ZeroMeansNoChange | 时间戳=0 表示不更新（R24） |
| Write_ImplicitTimestampUpdate | Write 后 LastWriteTime/ChangeTime 已更新（R24） |

---

## 3. 集成测试

```csharp
[Collection("WinFsp")]
public class WinFspComplianceTests : IAsyncLifetime
{
    private VMemFileSystem _fs;
    private string _mountPoint;

    public async Task InitializeAsync()
    {
        _fs = new VMemFileSystem(/* 1GB, 4KB pages */);
        _mountPoint = "T:\\";
        await _fs.MountAsync(_mountPoint);
    }

    [Fact] public void CreateFile_ReadBack() { /* File.WriteAllText → ReadAllText */ }
    [Fact] public void RenameFile_Atomic() { /* File.Move → exists at new, not at old */ }
    [Fact] public void DeleteFile_FreesMemory() { /* delete → pool AvailablePages increased */ }
    [Fact] public void ConcurrentReadWrite_8Threads() { /* parallel read/write, verify data integrity */ }
    [Fact] public void SecurityDescriptor_InheritedFromParent() { /* create subdir, check ACL */ }
    [Fact] public void DiskFull_ReturnsError() { /* fill disk to capacity, next write returns error */ }
    [Fact] public void MountUnmount_NoResourceLeak() { /* mount/unmount 10 cycles, memory stable */ }

    public async Task DisposeAsync() => await _fs.UnmountAsync();
}
```

### 缓存一致性测试

```csharp
[Fact]
public void Rename_ThenRead_ReturnsNewContent_WithKernelCache()
{
    // EnableKernelCache = true
    File.WriteAllText(@"T:\a.txt", "hello");
    File.Move(@"T:\a.txt", @"T:\b.txt");

    Assert.False(File.Exists(@"T:\a.txt"));           // 缓存已失效
    Assert.Equal("hello", File.ReadAllText(@"T:\b.txt"));
}
```

---

## 4. 性能基准

```csharp
[MemoryDiagnoser]
public class IoThroughputBenchmarks
{
    [Benchmark] public void SequentialWrite_1MB_Blocks() { /* ... */ }
    [Benchmark] public void SequentialRead_1MB_Blocks() { /* ... */ }
    [Benchmark] public void RandomWrite_4KB_Blocks() { /* ... */ }
    [Benchmark] public void RandomRead_4KB_Blocks() { /* ... */ }
}

[MemoryDiagnoser]
public class PagePoolBenchmarks
{
    [Benchmark] public void Rent_Return_SinglePage() { /* ... */ }
    [Benchmark] public void RentBatch_1000Pages() { /* ... */ }
    [Benchmark] public void Reserve_Unreserve_CAS() { /* ... */ }
}
```

**对比基准**：在相同硬件上运行 hooyao/RamDrive 基准测试，对比吞吐量。

---

## 5. 正确性不变量（辩论裁决 R2/R6：Release O(1) + 采样 O(n)）

| 不变量 | 表达式 | 检查时机 | Release 行为 |
|--------|--------|---------|-------------|
| **CommittedWithinCapacity** | `rentedCount + reservedCount <= maxPages` | 每次 Rent/Reserve | ✓ O(1)，违反→拒绝操作+ERR |
| **NoNegativeCount** | `rentedCount >= 0 && reservedCount >= 0` | 每次 Return/Unreserve | ✓ O(1)，违反→ERR |
| PoolConsistent | `rentedCount + freeCount <= allocatedCount` | 采样（每 1024 次）或 Dispose | O(1) 计数器版 |
| NoPageLeak | 所有已 Rent 的页在 pageTable 或 in-transit 中 | Dispose | ✓ 强制 Free |
| ReservationsConsistent | `Σ(file.reservedPages) == pool.reservedCount` | Dispose | ✓ ERR 日志 |
| NotifyAfterMutation | FspNotify 已调用（EnableKernelCache 时） | 回调出口 | ✓ O(1) flag |

> V1 废弃热路径 O(n) 检查（`freeStack.Count`、`PageTableIntegrity` 逐页扫描）。

---

## 6. CI 流水线（GitHub Actions）— L3 增强版

```yaml
name: CI
on:
  push:
  pull_request:
  schedule:
    - cron: '0 2 * * *'   # nightly stress + 全量合规

env:
  VMEM_LOG_PATH: ${{ github.workspace }}/test-logs

jobs:
  unit:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '9.0.x' }
      - run: dotnet test tests/VMem.Core.Tests -c Release --collect:"XPlat Code Coverage"

  integration:
    runs-on: windows-latest
    needs: unit
    steps:
      - uses: actions/checkout@v4
      - run: choco install winfsp -y
      - run: dotnet test tests/VMem.Integration.Tests -c Release --filter "Category!=FullCompliance"
      # PR: 跳过 winfsp-tests 全量

  compliance-full:
    if: github.ref == 'refs/heads/main' || github.event_name == 'schedule'
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - run: choco install winfsp -y
      - run: dotnet test tests/VMem.Integration.Tests -c Release --filter "Category=FullCompliance"
      - name: Check Contract Violations
        if: always()
        shell: pwsh
        run: |
          $v = Get-ChildItem "$env:VMEM_LOG_PATH/*.json" -EA SilentlyContinue |
            % { Get-Content $_ | ConvertFrom-Json } |
            ? { $_.msg -eq "CONTRACT VIOLATION" }
          if ($v) { exit 1 }
      - name: Upload Test Logs
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-logs-${{ github.sha }}
          path: |
            ${{ env.VMEM_LOG_PATH }}/*.json
            **/*.trx
          retention-days: 14

  stress-nightly:
    if: github.event_name == 'schedule'
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - run: dotnet test tests/VMem.Stress.Tests -c Release --filter "Category=Stress"

  aot-gate:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '9.0.x' }
      - run: dotnet publish src/VMem.Cli -c Release -r win-x64 /p:PublishAot=true
        # R18: Phase 1 即加 AOT 门禁；PR=build, main=publish

  benchmark:
    if: github.ref == 'refs/heads/main'
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - run: dotnet run -c Release --project tests/VMem.Benchmarks
      - uses: actions/upload-artifact@v4
        with:
          name: benchmarks-${{ github.sha }}
          path: BenchmarkDotNet.Artifacts/
          retention-days: 30
```

**CI 注意点（辩论裁决 R9）**：
- 挂载用随机盘符（如 `Z:`）+ `IClassFixture` 串行，避免并行 job 冲突
- winfsp-tests 通过 `Process.Start` 调用 WinFsp 安装目录下的 `winfsp-tests.exe`
- Benchmark **不设 hard gate**（不因 ±10% 失败），结果存 artifact 30 天

---

## 7. L3 AI 范式测试增强

> 详见 [子方案 ⑦ - 日志与可观测性](./vmem_07_logging_observability.plan.md)

### 7.1 日志驱动的自动化回归测试

当 AI 通过 Replay Log 定位到 bug 后，可自动从回放日志生成回归测试用例：

```csharp
/// 从 replay log JSONL 文件生成 xUnit 测试类
public static class ReplayTestGenerator
{
    public static string GenerateTestClass(string replayLogPath)
    {
        var operations = File.ReadLines(replayLogPath)
            .Select(line => JsonSerializer.Deserialize<ReplayEntry>(line))
            .Where(e => e?.Msg == "FS.Replay")
            .ToList();

        var sb = new StringBuilder();
        sb.AppendLine("[Collection(\"Replay\")]");
        sb.AppendLine("public class ReplayTests : IAsyncLifetime { ... }");

        foreach (var op in operations)
        {
            sb.AppendLine($"    [Fact] public void Replay_Seq{op.Props.Seq}_{op.Props.Op}()");
            sb.AppendLine($"    {{ /* auto-generated from replay log */ }}");
        }

        return sb.ToString();
    }
}
```

### 7.2 测试中的日志验证

测试可断言日志中不包含契约违反和未预期错误：

```csharp
public class LogAssertFixture : IDisposable
{
    private readonly List<LogEvent> _events = new();

    public LogAssertFixture()
    {
        // 注入内存 Sink 捕获所有日志
        Log.Logger = new LoggerConfiguration()
            .WriteTo.Sink(new InMemorySink(_events))
            .CreateLogger();
    }

    public void AssertNoContractViolations()
    {
        var violations = _events.Where(e => e.MessageTemplate.Text.Contains("CONTRACT VIOLATION"));
        Assert.Empty(violations);
    }

    public void AssertNoErrors()
    {
        var errors = _events.Where(e => e.Level >= LogEventLevel.Error);
        Assert.Empty(errors);
    }

    public void AssertContains(LogEventLevel level, string messageFragment)
    {
        Assert.Contains(_events, e =>
            e.Level == level && e.RenderMessage().Contains(messageFragment));
    }
}
```

### 7.3 CI 契约违反自动拦截

CI 流水线在测试完成后自动解析 JSON 日志，若检测到 `CONTRACT VIOLATION` 条目则标记构建失败（见上方 CI YAML 中的 `Check Contract Violations` 步骤）。

### 7.4 故障注入框架（R71-R80 四巨头审查）

> **Google 总监**：生产系统需要可编程故障注入验证降级路径。
> **Amazon 总监**：所有异常路径必须有自动化覆盖。

```csharp
/// V1：编译时开关的轻量故障注入（不引入运行时开销）
public static class FaultInjection
{
#if FAULT_INJECTION
    [ThreadStatic] private static FaultConfig? t_config;

    public static void Configure(FaultConfig config) => t_config = config;
    public static void Clear() => t_config = null;

    /// 在 NativeMemory.AllocZeroed 前调用
    public static bool ShouldFailAlloc()
        => t_config?.AllocFailRate > 0 && Random.Shared.NextDouble() < t_config.AllocFailRate;

    /// 在 CAS 操作前注入延迟
    public static void MaybeDelay()
    {
        if (t_config?.CasDelayMs > 0) Thread.Sleep(t_config.CasDelayMs);
    }
#else
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool ShouldFailAlloc() => false;
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void MaybeDelay() { }
#endif
}

public record FaultConfig(
    double AllocFailRate = 0,      // 0~1, 分配失败概率
    int CasDelayMs = 0,            // CAS 操作前延迟（模拟高竞争）
    double PipeDisconnectRate = 0  // IPC 断连概率
);
```

**测试用例**（编译时 `FAULT_INJECTION` 宏）：

| 测试 | 故障 | 验证点 |
|------|------|--------|
| Write_AllocFail_MidBatch_RollsBack | AllocFailRate=0.5 | 部分分配失败后 BatchLease 归还所有已分配页 |
| TryReserve_HighContention_EventuallyConverges | CasDelayMs=10 | 8 线程并发 Reserve 最终全部完成/拒绝 |
| IpcClient_Disconnect_Reconnects | PipeDisconnectRate=0.3 | 管道断开后客户端成功重连 |

### 7.5 模糊测试策略（V2 目标）

> **Oracle 总监**：文件系统必须能抵御任意畸形输入。

| 模糊目标 | 工具 | 输入空间 | V1 | V2 |
|----------|------|---------|-----|-----|
| IPC JSON 解析 | SharpFuzz | 畸形 JSON + 超大 payload | ✗ | ✓ |
| FS 路径处理 | 自定义 | 空路径/超长/NUL字符/特殊字符 | 部分（边界测试） | ✓ |
| Read/Write 偏移 | 自定义 | offset<0, offset>long.Max/2, length=0 | 部分（单元测试） | ✓ |
| SecurityDescriptor | SharpFuzz | 畸形 SD 二进制 | ✗ | ✓ |

### 7.6 测试覆盖率与日志覆盖率

| 指标 | 目标 | 检查方式 |
|------|------|---------|
| 代码行覆盖率 | ≥ 80% | `dotnet test --collect:"XPlat Code Coverage"` |
| 分支覆盖率 | ≥ 70% | Coverlet |
| **日志覆盖率** | 每个 WinFsp 回调有入口+出口日志 | 自定义 Roslyn Analyzer（可选） |
| **契约覆盖率** | 每个关键方法有 Requires/Ensures | Code Review 检查 |
