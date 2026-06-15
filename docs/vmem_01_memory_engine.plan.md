---
name: "VMem 子方案 ① - 内存引擎"
overview: "PagePool 页池、PageLease 安全租借、容量预留系统、StatsCollector 实时统计。核心目标：零 GC 压力、无锁高并发、防泄漏。"
todos:
  - id: pagepool-core
    content: "实现 PagePool：NativeMemory + ConcurrentStack + 惰性/预分配策略"
    status: pending
  - id: pagepool-reserve
    content: "实现容量预留：TryReserve / Unreserve / TryRentFromReservation + CAS 循环"
    status: pending
  - id: pagelease
    content: "实现 PageLease ref struct：Commit/Dispose 配对、异常自动归还"
    status: pending
  - id: stats
    content: "实现 StatsCollector：Interlocked 累加、GetAndResetSnapshot"
    status: pending
  - id: unit-tests
    content: "PagePool 单元测试：分配/归还/耗尽/预留竞态/泄漏检测/清零验证"
    status: pending
isProject: false
---

# 子方案 ① - 内存引擎

> 父方案：[VMem RAM Disk Design](./vmem_ram_disk_design_031231c8.plan.md)
> 对应代码：`src/VMem.Core/Memory/`
> Phase：1

---

## 1. PagePool - 页池

### 1.1 API 设计

```csharp
public sealed class PagePool : IDisposable
{
    private readonly ConcurrentStack<nint> _freePages;
    private readonly int _pageSize;          // 默认 4KB，可选 64KB
    private readonly long _maxPages;         // 页数上限 = CapacityBytes / PageSize
    private long _allocatedCount;            // 已从 OS 分配的总页数（Interlocked）
    private long _rentedCount;               // 当前被文件占用的页数（Interlocked）
    private long _reservedCount;             // SetLength 预留的页数（Interlocked）
    private int _disposed;

    // === 基本操作 ===
    public bool TryRent(out nint page);
    public int TryRentBatch(Span<nint> pages);
    public void Return(nint page);           // 自动 NativeMemory.Clear 清零
    public void ReturnBatch(ReadOnlySpan<nint> pages);

    // === 容量预留 ===
    public bool TryReserve(long pageCount);
    public void Unreserve(long pageCount);
    public bool TryRentFromReservation(out nint page);

    // === 统计 ===
    public long AllocatedPages => Volatile.Read(ref _allocatedCount);
    public long RentedPages => Volatile.Read(ref _rentedCount);
    public long ReservedPages => Volatile.Read(ref _reservedCount);
    public long AvailablePages => _maxPages - RentedPages - ReservedPages;
    public int PageSize => _pageSize;
}
```

### 1.2 页大小策略

| 页大小 | 适用场景 | 内部碎片 |
|--------|----------|----------|
| **4KB（默认）** | TEMP/浏览器缓存/小文件密集 | 1 字节文件浪费 4095 B |
| 64KB | 大文件/媒体缓存 | 1 字节文件浪费 65535 B |

hooyao/RamDrive 默认 64KB 面向大文件；本项目面向通用场景，4KB 对齐 OS 页大小更合适。

### 1.3 容量预留系统

**问题场景**（学习自 hooyao/RamDrive PR #8）：
```
16GB 磁盘 → 10 个文件各 SetLength(2GB) → 逻辑总大小 20GB > 物理 16GB
→ 后续 Write 时无法检测超额（页是按需分配的）
```

**解决方案**：
- `SetLength(newSize)` 扩展时：`TryReserve(additionalPages)` CAS 原子扣减可用配额
- `Write` 分配页面时：优先用 `TryRentFromReservation()`，已预留不再检查容量
- `SetLength(smaller)` 截断 / 文件删除时：`Unreserve()` 归还配额

**CAS 循环伪码**：
```csharp
public bool TryReserve(long pageCount)
{
    while (true)
    {
        long current = Volatile.Read(ref _reservedCount);
        long rented = Volatile.Read(ref _rentedCount);
        if (rented + current + pageCount > _maxPages)
            return false; // STATUS_DISK_FULL
        if (Interlocked.CompareExchange(ref _reservedCount, current + pageCount, current) == current)
            return true;
    }
}
```

### 1.4 内存分配策略

- **惰性分配（默认）**：`TryRent` 时 free stack 空 → `NativeMemory.AllocZeroed` 分配新页 → Interlocked 递增 `_allocatedCount` → 入 free stack → 再 pop
- **预分配（`PreAllocate=true`）**：构造时一次性分配 `_maxPages` 个页面，全部入 free stack。启动慢但运行时无页错误延迟
- 达到 `_maxPages` 上限后停止分配 → `TryRent` 返回 `false`

### 1.5 安全清零

`Return` 时调用 `NativeMemory.Clear(page, pageSize)` 归零页面。防止：
- 文件 A 删除后页面被文件 B 复用 → B 读到 A 的残留数据
- 安全敏感数据（如加密密钥）残留在已释放的页中

---

## 2. PageLease - 安全租借

```csharp
public ref struct PageLease
{
    private readonly PagePool _pool;
    private nint _page;
    private bool _committed;

    internal PageLease(PagePool pool, nint page) { _pool = pool; _page = page; }
    public nint Page => _page;

    /// 移交所有权给调用方（如 PagedFileContent），Dispose 不再归还。
    public nint Commit() { _committed = true; return _page; }

    public void Dispose() { if (!_committed && _page != 0) _pool.Return(_page); }
}
```

**使用模式**：
```csharp
if (!_pool.TryRent(out var raw)) return NtStatus.STATUS_DISK_FULL;
using var lease = new PageLease(_pool, raw);

// ... 可能抛异常的初始化操作 ...

_pageTable[index] = lease.Commit();  // 成功后移交所有权
```

**设计要点**：
- `ref struct` 不可装箱、不可逃逸 → 保证作用域内使用
- 异常路径自动 `Dispose` → 归还页面，不泄漏

**批量模式**：

> ⚠ `ref struct` 不能作为泛型类型参数，因此 `Span<PageLease>` 是**非法的**。
> 批量分配使用独立的 `BatchLease` 类（非 ref struct）来管理：

```csharp
public sealed class BatchLease : IDisposable
{
    private readonly PagePool _pool;
    private readonly nint[] _pages;
    private readonly bool[] _committed;
    private int _count;
    private bool _disposed;

    internal BatchLease(PagePool pool, int maxCount) { ... }

    public int Count => _count;
    public nint this[int index] => _pages[index];

    /// 批量从池中分配，返回实际分配数
    internal int RentBatch(int requested) { ... }

    /// 移交单页所有权
    public nint Commit(int index) { _committed[index] = true; return _pages[index]; }

    /// 移交全部已分配页的所有权
    public nint[] CommitAll() { ... }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        for (int i = 0; i < _count; i++)
            if (!_committed[i]) _pool.Return(_pages[i]);
    }
}
```

使用模式（三阶段写入 Phase 2）：
```csharp
using var batch = _pool.RentBatch(missingPageCount);
if (batch.Count < missingPageCount)
    return NtStatus.STATUS_DISK_FULL;

// Phase 3: 写锁内逐页 Commit
for (int i = 0; i < batch.Count; i++)
{
    if (_pageTable[targetIndex] == 0)
        _pageTable[targetIndex] = batch.Commit(i);
    // else: 已被并发写入填充 → 不 Commit → Dispose 自动归还
}
```

---

## 3. StatsCollector - 实时统计

> **性能设计**：WinFsp 回调在多线程中并发执行。如果所有线程对同一个 `long` 做 `Interlocked.Add`，
> 会产生**缓存行伪共享（false sharing）**，显著降低吞吐量。
> 方案：使用 `[ThreadStatic]` 线程本地累加 + 定时聚合，消除热路径上的原子操作。

```csharp
public sealed class StatsCollector
{
    // 线程本地计数器（热路径零竞争）
    [ThreadStatic] private static ThreadLocalCounters? t_counters;

    private readonly ConcurrentBag<ThreadLocalCounters> _allCounters = new();
    private readonly Stopwatch _uptime;

    private sealed class ThreadLocalCounters
    {
        public long ReadBytes;
        public long WriteBytes;
        public long ReadOps;
        public long WriteOps;
    }

    private ThreadLocalCounters GetCounters()
    {
        var c = t_counters;
        if (c is null)
        {
            c = new ThreadLocalCounters();
            t_counters = c;
            _allCounters.Add(c);
        }
        return c;
    }

    public void RecordRead(int bytes)
    {
        var c = GetCounters();
        c.ReadBytes += bytes;    // 线程本地，无需原子操作
        c.ReadOps++;
    }

    public void RecordWrite(int bytes)
    {
        var c = GetCounters();
        c.WriteBytes += bytes;
        c.WriteOps++;
    }

    /// 获取快照并重置速率计数器（GUI 轮询，默认 1 秒间隔）。
    /// 聚合所有线程本地计数器 → 仅在读取路径（1次/秒）产生竞争。
    public IoSnapshot GetAndResetSnapshot();
}

public readonly record struct IoSnapshot(
    long ReadBytesPerSec, long WriteBytesPerSec,
    long ReadOpsPerSec, long WriteOpsPerSec,
    long TotalUsedBytes, long TotalReservedBytes, long TotalCapacityBytes,
    int FileCount, TimeSpan Uptime);
```

---

## 4. 正确性不变量（L3 契约化检查）

参考 hooyao/RamDrive TLA+ 验证。**与传统做法不同，所有不变量在 Release 构建中也生效**，违反时通过 `Contract.Invariant` 写入结构化日志（见[子方案 ⑦](./vmem_07_logging_observability.plan.md)），不崩溃但记录完整上下文供 AI 分析：

| 不变量 | 表达式 | 检查时机 |
|--------|--------|---------|
| PoolConsistent | `rentedCount + freeStack.Count <= allocatedCount <= maxPages` | 每次 Rent/Return 后 |
| NoPageLeak | 所有已 Rent 的页在某文件 pageTable 或 BatchLease in-transit 中 | Dispose 时 → **强制 NativeMemory.Free 泄漏页** |
| ReservationsConsistent | `Σ(file.reservedPages) == pool.reservedCount` | Dispose 时 |
| CommittedWithinCapacity | `rentedCount + reservedCount <= maxPages` | 每次 Rent/Reserve 后 |
| NoNegativeCount | `rentedCount >= 0 && reservedCount >= 0` | 每次 Return/Unreserve 后 |

---

## 5. 日志与可观测性（L3 范式补丁）

> 详见 [子方案 ⑦ - 日志与可观测性](./vmem_07_logging_observability.plan.md)

### 5.1 PagePool 操作日志

| 操作 | 日志级别 | 记录内容 |
|------|---------|---------|
| `TryRent` 成功 | `DBG` | correlationId, 分配的页地址, 当前 rentedCount/availablePages |
| `TryRent` 失败（池耗尽） | `WRN` | correlationId, 请求页数, maxPages, 当前 rentedCount/reservedCount |
| `TryRentBatch` | `DBG` | correlationId, 请求页数, 实际分配数, 耗时(μs) |
| `Return` | `DBG` | correlationId, 归还页地址, 归还后 rentedCount |
| `TryReserve` 成功 | `DBG` | correlationId, 预留页数, 新 reservedCount |
| `TryReserve` 失败 | `WRN` | correlationId, 请求页数, 剩余容量 |
| NativeMemory.AllocZeroed | `DBG` | 新页物理分配, allocatedCount 变化 |
| 构造完成 | `INF` | maxPages, pageSize, preAllocate, 总容量(MB) |
| Dispose | `INF` | allocatedCount, rentedCount(泄漏数), freeStack.Count |
| Dispose 检测到泄漏 | `ERR` | 泄漏页数, 泄漏占比 |

### 5.2 性能追踪

```csharp
public bool TryRent(out nint page)
{
    var sw = Stopwatch.StartNew();
    // ... 分配逻辑 ...
    sw.Stop();
    if (sw.Elapsed.TotalMilliseconds > 1.0)
    {
        _logger.Warning("SLOW OPERATION TryRent took {ElapsedMs}ms, " +
            "likely OS page fault during NativeMemory.AllocZeroed",
            sw.Elapsed.TotalMilliseconds);
    }
}
```

| 操作 | 慢操作阈值 | 可能原因 |
|------|-----------|---------|
| `TryRent` | > 1ms | OS 页错误（惰性分配首次触碰物理页） |
| `TryRentBatch` | > 5ms (1000页) | 连续 OS 页错误 |
| `Return` + Clear | > 0.5ms | 大页 (64KB) 清零开销 |

### 5.3 StatsCollector 周期摘要

`StatsCollector.GetAndResetSnapshot()` 每 60 秒输出一次 `INF` 级别的统计摘要日志，包含吞吐量、IOPS、池利用率、文件数、契约违反计数等（格式见子方案 ⑦ §6.2）。
