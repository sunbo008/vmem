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
    // V1: ConcurrentStack<nint>（简洁可靠）
    // R42 注意：每 Push/Pop 创建 Node 对象，百万页面时产生 GC 压力
    // V2 备选：无 GC 的 lock-free stack（NativeMemory 链表或 mpsc queue）
    private readonly ConcurrentStack<nint> _freePages;
    private readonly int _pageSize;          // 默认 4KB，可选 64KB
    private readonly long _maxPages;         // 页数上限 = CapacityBytes / PageSize
    private long _allocatedCount;            // 已从 OS 分配的总页数（Interlocked）
    private long _rentedCount;               // 当前被文件占用的页数（Interlocked）
    private long _reservedCount;             // SetLength 预留的页数（Interlocked）
    private int _disposed;

    // === 基本操作 ===
    public bool TryRent(out nint page)
    {
        page = 0;
        if (Volatile.Read(ref _disposed) != 0) return false;

        // 容量快速检查
        if (Volatile.Read(ref _rentedCount) + Volatile.Read(ref _reservedCount) >= _maxPages)
            return false;

        if (!_freePages.TryPop(out page))
        {
            // free stack 空 → 尝试从 OS 分配新页
            if (Volatile.Read(ref _allocatedCount) >= _maxPages) return false;
            try
            {
                page = (nint)NativeMemory.AllocZeroed((nuint)_pageSize);
            }
            catch (OutOfMemoryException)
            {
                _logger.Error("OOM: AllocZeroed failed, allocated={Allocated}", _allocatedCount);
                return false;
            }
            Interlocked.Increment(ref _allocatedCount);
        }

        Interlocked.Increment(ref _rentedCount);
        Interlocked.Decrement(ref _freeCount);
        return true;
    }
    public int TryRentBatch(Span<nint> pages);
    // Amazon 审查：Return 须检查 _disposed（延迟归还场景）和页面有效性
    public void Return(nint page)
    {
        if (page == 0) throw new ArgumentException("Cannot return null page");
        NativeMemory.Clear((void*)page, (nuint)_pageSize);
        _freePages.Push(page);
        Interlocked.Decrement(ref _rentedCount);
        Interlocked.Increment(ref _freeCount);
        // 不检查 _disposed：Dispose 后的延迟 Return 仍需归还以免泄漏
        // Dispose 会在最后统一 Free 所有页面
    }
    public void ReturnBatch(ReadOnlySpan<nint> pages);

    // === 容量预留 ===
    public bool TryReserve(long pageCount);
    public void Unreserve(long pageCount);
    public bool TryRentFromReservation(out nint page)
    {
        // 从已预留配额中分配物理页（不再检查总容量，预留时已检查）
        page = 0;
        // 1. 先扣减 reserved → 增加 rented（保持 total 不变）
        while (true)
        {
            long currentReserved = Volatile.Read(ref _reservedCount);
            if (currentReserved <= 0) return false; // 无可用预留
            if (Interlocked.CompareExchange(ref _reservedCount,
                    currentReserved - 1, currentReserved) == currentReserved)
                break;
        }
        // 2. 分配物理页
        if (!_freePages.TryPop(out page))
        {
            try { page = (nint)NativeMemory.AllocZeroed((nuint)_pageSize); }
            catch (OutOfMemoryException)
            {
                Interlocked.Increment(ref _reservedCount); // 回退预留
                return false;
            }
            Interlocked.Increment(ref _allocatedCount);
        }
        Interlocked.Increment(ref _rentedCount);
        Interlocked.Decrement(ref _freeCount);
        return true;
    }

    // === 统计 ===
    public long AllocatedPages => Volatile.Read(ref _allocatedCount);
    public long RentedPages => Volatile.Read(ref _rentedCount);
    public long ReservedPages => Volatile.Read(ref _reservedCount);
    public long AvailablePages => _maxPages - RentedPages - ReservedPages;
    public int PageSize => _pageSize;
    public long TotalCapacityBytes => _maxPages * _pageSize;

    public void Dispose()
    {
        if (Interlocked.Exchange(ref _disposed, 1) != 0) return; // 防双释放

        long leakedPages = Volatile.Read(ref _rentedCount);
        if (leakedPages > 0)
        {
            _logger.Error("PagePool LEAK: {LeakedPages} pages not returned ({LeakedBytes} bytes)",
                leakedPages, leakedPages * _pageSize);
            Debug.Fail($"PagePool leak: {leakedPages} pages");
        }

        long freedCount = 0;
        while (_freePages.TryPop(out nint page))
        {
            NativeMemory.Free((void*)page);
            freedCount++;
        }

        _logger.Information("PagePool disposed: freed={Freed} allocated={Allocated} leaked={Leaked}",
            freedCount, Volatile.Read(ref _allocatedCount), leakedPages);
    }
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
    if (pageCount <= 0) throw new ArgumentOutOfRangeException(nameof(pageCount));
    if (Volatile.Read(ref _disposed) != 0) throw new ObjectDisposedException(nameof(PagePool));

    while (true)
    {
        // Google 审查：rented + reserved 非原子快照存在 TOCTOU 窗口
        // 修复：在 CAS 成功后做 post-condition 校验，超额则回退
        long currentReserved = Volatile.Read(ref _reservedCount);
        long currentRented = Volatile.Read(ref _rentedCount);
        long total = currentRented + currentReserved + pageCount;
        if (total > _maxPages)
            return false; // STATUS_DISK_FULL

        if (Interlocked.CompareExchange(ref _reservedCount,
                currentReserved + pageCount, currentReserved) == currentReserved)
        {
            // 后置校验：如果 CAS 成功但实际已超额（极端并发），回退
            if (Volatile.Read(ref _rentedCount) + Volatile.Read(ref _reservedCount) > _maxPages)
            {
                Interlocked.Add(ref _reservedCount, -pageCount); // 回退
                return false;
            }
            return true;
        }
        // CAS 失败 → 自旋重试
    }
}
```

### 1.4 内存分配策略

- **惰性分配（默认）**：`TryRent` 时 free stack 空 → `NativeMemory.AllocZeroed` 分配新页 → Interlocked 递增 `_allocatedCount` → push to free stack → pop
- **预分配（`PreAllocate=true`）**：构造时一次性分配 `_maxPages` 个页面，全部入 free stack。启动慢但运行时无页错误延迟
- 达到 `_maxPages` 上限后停止分配 → `TryRent` 返回 `false`
- **OOM 处理（R41）**：`NativeMemory.AllocZeroed` 抛 `OutOfMemoryException` → `TryRent` 捕获并返回 false（映射 STATUS_DISK_FULL）；ERR 日志记录 allocatedCount + requestedSize

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

    internal PageLease(PagePool pool, nint page)
    {
        Debug.Assert(page != 0, "PageLease created with null page");
        _pool = pool;
        _page = page;
    }

    public nint Page => _page != 0 ? _page : throw new ObjectDisposedException(nameof(PageLease));

    /// 移交所有权给调用方（如 PagedFileContent），Dispose 不再归还。
    public nint Commit()
    {
        if (_committed) throw new InvalidOperationException("Already committed");
        if (_page == 0) throw new ObjectDisposedException(nameof(PageLease));
        _committed = true;
        return _page;
    }

    public void Dispose()
    {
        if (!_committed && _page != 0)
        {
            _pool.Return(_page);
            _page = 0; // 防止 double-return
        }
    }
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

    /// 移交单页所有权（Oracle 审查：增加边界检查 + Dispose 后防护）
    public nint Commit(int index)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        ArgumentOutOfRangeException.ThrowIfNegative(index);
        ArgumentOutOfRangeException.ThrowIfGreaterThanOrEqual(index, _count);
        if (_committed[index]) throw new InvalidOperationException($"Page {index} already committed");
        _committed[index] = true;
        return _pages[index];
    }

    /// 移交全部已分配页的所有权
    public nint[] CommitAll()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        for (int i = 0; i < _count; i++) _committed[i] = true;
        return _pages[.._count];
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        for (int i = 0; i < _count; i++)
            if (!_committed[i] && _pages[i] != 0) _pool.Return(_pages[i]);
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

> **V1/V2 分级设计（辩论裁决 R6）**：
> - **V1**：池级指标 `Volatile.Read`（零热路径竞争）；IO 统计默认关或 padded `Interlocked`
> - **V2**（profile 触发）：`[ThreadStatic]` 线程本地累加 + 1s 聚合

```csharp
public sealed class StatsCollector
{
    private readonly Stopwatch _uptime;

    // V1 池级指标（直接读 PagePool 字段，零额外竞争）
    // GUI 1 秒轮询 GetSnapshot() 读取
    public IoSnapshot GetSnapshot();

    // V1 可选：IO 统计（padded Interlocked，默认关闭）
    [StructLayout(LayoutKind.Explicit, Size = 128)]
    private struct PaddedCounter { [FieldOffset(64)] public long Value; }

    private PaddedCounter _readBytes;
    private PaddedCounter _writeBytes;

    public void RecordRead(int bytes)
    {
        Interlocked.Add(ref _readBytes.Value, bytes);
    }

    public void RecordWrite(int bytes)
    {
        Interlocked.Add(ref _writeBytes.Value, bytes);
    }

    // V2 触发条件：8T 压测 stats 占 >5% cycles → 切 [ThreadStatic] + 1s GetAndResetSnapshot
}

public readonly record struct IoSnapshot(
    long TotalUsedBytes, long TotalReservedBytes, long TotalCapacityBytes,
    int FileCount, TimeSpan Uptime,
    double PoolUtilizationPercent,    // V1 新增：Pool 利用率
    long ReadBytesTotal, long WriteBytesTotal);
```

---

## 4. 正确性不变量（辩论裁决 R2/R6：Release O(1) + 采样 O(n)）

参考 hooyao/RamDrive TLA+ 验证。

> **核心变更**：审查 P0 指出 `freeStack.Count` 是 O(n) 操作，不能在热路径 Release 检查。
> V1 策略：**Release 仅 O(1) 硬边界**；O(n) 完整性改为**采样（每 1024 次）或 Dispose 时**。
> O(1) 边界违反 → **拒绝当前操作** + ERR 结构化日志（含 correlationId）；不 silent continue。

| 不变量 | 表达式 | 检查时机 | Release 行为 |
|--------|--------|---------|-------------|
| **CommittedWithinCapacity** | `rentedCount + reservedCount <= maxPages` | 每次 Rent/Reserve 后 | ✓ O(1)，违反→拒绝操作 |
| **NoNegativeCount** | `rentedCount >= 0 && reservedCount >= 0` | 每次 Return/Unreserve 后 | ✓ O(1)，违反→ERR 日志 |
| PoolConsistent | `rentedCount + freeCount <= allocatedCount` | **采样（每 1024 次）** 或 Dispose 时 | O(1) 计数器版本 |
| NoPageLeak | 所有已 Rent 的页在某文件 pageTable 或 BatchLease 中 | Dispose 时 | ✓ 强制 Free 泄漏页 |
| ReservationsConsistent | `Σ(file.reservedPages) == pool.reservedCount` | Dispose 时 | ✓ ERR 日志 |
| PageTableIntegrity | 所有非零页表项指向有效 NativeMemory 地址 | Dispose 时（O(n) 扫描） | ✓ Debug 全量 |

> V1 维护 `_freeCount` 整数计数器替代 `freeStack.Count`，Rent 时递减、Return 时递增，O(1)。

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

### 5.2 性能追踪（辩论裁决 R6：V1 热路径零计时）

> **V1**：Release 热路径**不内嵌任何 Stopwatch/ValueStopwatch**。
> 性能数据通过 **BenchmarkDotNet**（外置）和 **ETW/EventPipe** 获取。
> Debug 构建可选 `#if DEBUG` 包裹的 ValueStopwatch 分阶段计时。

```csharp
// V1 Release：零计时
public bool TryRent(out nint page)
{
    // 纯分配逻辑，无 Stopwatch
}

// V1 Debug（可选）：
#if DEBUG
public bool TryRent(out nint page)
{
    var start = Stopwatch.GetTimestamp();
    // ... 分配逻辑 ...
    var elapsed = Stopwatch.GetElapsedTime(start);
    if (elapsed.TotalMilliseconds > 1.0)
        _logger.Warning("SLOW TryRent {ElapsedMs}ms", elapsed.TotalMilliseconds);
}
#endif
```

### 5.3 StatsCollector 周期摘要（V1 不做，V2 触发）

> V1 不做 60s 周期统计摘要。池利用率通过 `IoSnapshot.PoolUtilizationPercent` 按需查询。
