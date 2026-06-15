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

## 1. 分层测试体系

| 层级 | 框架 | 覆盖目标 | 运行频率 |
|------|------|----------|---------|
| 单元测试 | xUnit | PagePool（含 Reserve CAS）、PagedFileContent（三阶段/TOCTOU）、DirectoryTree、SD | 每次提交 |
| 集成测试 | xUnit | 挂载真实 WinFsp FS，文件 CRUD/权限/并发 | 每次提交（需 WinFsp） |
| 合规测试 | winfsp-tests | WinFsp 官方套件，验证 POSIX/Win32 语义 | 每次提交 |
| 缓存一致性 | 自定义 | `EnableKernelCache=true` 下 Rename→Read、Delete→Exists | 每次提交 |
| 压力测试 | 自定义 | 8 线程并发读写 + Rename/Delete + SetLength，10 分钟 | 每日/手动 |
| 性能基准 | BenchmarkDotNet | 顺序/随机读写吞吐、PagePool 分配延迟 | main 分支合并 |
| 泄漏检测 | 自定义 | 循环创建/删除 10 万文件后 RentedPages + ReservedPages == 0 | 每日/手动 |

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
| Dispose_DetectsLeak | 有未归还页时 Debug 断言 |
| LazyAllocation_AllocsOnDemand | 首次 Rent 才触发 NativeMemory.Alloc |
| PreAllocate_AllocsAll | PreAllocate=true 构造时全部分配 |

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

### DirectoryTreeTests

| 测试 | 验证点 |
|------|--------|
| Lookup_CaseInsensitive | 路径大小写不敏感 |
| TryCreate_ConcurrentSameName | 并发创建同名只有一个成功 |
| Rename_CrossDirectory_NoDeadlock | 跨目录 Rename 不死锁 |
| Rename_ReplaceExisting | replaceIfExists 正确替换 |
| TryRemove_NonEmptyDir_Fails | 非空目录删除失败 |
| EnumerateDirectory_SnapshotIsolation | 枚举期间并发修改不影响结果 |

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

## 5. 正确性不变量（Release 也生效）

**L3 范式变更**：不变量通过 `Contract.Invariant` 实现，**Release 构建中也检查**，违反时写入 `ERR` 级别结构化日志而非崩溃（详见 [子方案 ⑦](./vmem_07_logging_observability.plan.md) §4）：

| 不变量 | 表达式 | 检查时机 |
|--------|--------|---------|
| PoolConsistent | `rentedCount + freeStack.Count <= allocatedCount <= maxPages` | 每次 Rent/Return |
| NoPageLeak | 所有已 Rent 的页在某文件 pageTable 或 in-transit 中 | Dispose |
| ReservationsConsistent | `Σ(file.reservedPages) == pool.reservedCount` | Dispose |
| CommittedWithinCapacity | `rentedCount + reservedCount <= maxPages` | 每次 Rent/Reserve |
| PageTableIntegrity | 所有非零页表项指向有效 NativeMemory 地址 | Write Phase 3 后 |
| NotifyAfterMutation | Rename/Delete/Overwrite 后 FspNotify 已调用 | 回调出口 |

---

## 6. CI 流水线（GitHub Actions）— L3 增强版

```yaml
name: CI
on: [push, pull_request]

env:
  VMEM_LOG_PATH: ${{ github.workspace }}/test-logs

jobs:
  build-test:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install WinFsp
        run: choco install winfsp -y

      - name: Build
        run: dotnet build -c Release

      - name: Unit Tests
        run: dotnet test tests/VMem.Core.Tests -c Release --logger "trx;LogFileName=unit.trx"

      - name: Integration Tests
        run: dotnet test tests/VMem.Integration.Tests -c Release --logger "trx;LogFileName=integration.trx"

      # L3: 解析 JSON 日志，检查契约违反
      - name: Check Contract Violations
        if: always()
        shell: pwsh
        run: |
          $violations = Get-ChildItem "$env:VMEM_LOG_PATH/*.json" -ErrorAction SilentlyContinue |
            ForEach-Object { Get-Content $_ } |
            ConvertFrom-Json |
            Where-Object { $_.msg -eq "CONTRACT VIOLATION" }
          if ($violations.Count -gt 0) {
            Write-Error "Found $($violations.Count) contract violations!"
            $violations | ConvertTo-Json | Write-Output
            exit 1
          }

      # L3: 上传日志归档供 AI 离线分析
      - name: Upload Test Logs
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-logs-${{ github.sha }}
          path: |
            ${{ env.VMEM_LOG_PATH }}/*.json
            ${{ env.VMEM_LOG_PATH }}/*.log
            **/*.trx
          retention-days: 14

  benchmark:
    if: github.ref == 'refs/heads/main'
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install WinFsp
        run: choco install winfsp -y
      - name: Run Benchmarks
        run: dotnet run -c Release --project tests/VMem.Benchmarks
      - name: Upload Benchmark Results
        uses: actions/upload-artifact@v4
        with:
          name: benchmarks-${{ github.sha }}
          path: BenchmarkDotNet.Artifacts/
          retention-days: 30
```

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

### 7.4 测试覆盖率与日志覆盖率

| 指标 | 目标 | 检查方式 |
|------|------|---------|
| 代码行覆盖率 | ≥ 80% | `dotnet test --collect:"XPlat Code Coverage"` |
| 分支覆盖率 | ≥ 70% | Coverlet |
| **日志覆盖率** | 每个 WinFsp 回调有入口+出口日志 | 自定义 Roslyn Analyzer（可选） |
| **契约覆盖率** | 每个关键方法有 Requires/Ensures | Code Review 检查 |
