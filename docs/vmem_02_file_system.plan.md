---
name: "VMem 子方案 ② - 文件系统与 WinFsp 集成"
overview: "VMemFileSystem（IFileSystem 全部回调）、DirectoryTree、FileNode、PagedFileContent 三阶段写入、安全模型（SecurityDescriptor）、WinFsp 内核缓存 + Notify。"
todos:
  - id: filenode
    content: "实现 FileNode：元数据 + SecurityDescriptor + 引用计数 + AllocationSize"
    status: pending
  - id: directory-tree
    content: "实现 DirectoryTree：ConcurrentDictionary + 原子 Rename + 字典序加锁"
    status: pending
  - id: paged-content
    content: "实现 PagedFileContent：三阶段写入 + 稀疏读 + SetLength 预留"
    status: pending
  - id: vfs-callbacks
    content: "实现 VMemFileSystem 全部 WinFsp 回调（17 个必要 + 可选）"
    status: pending
  - id: security
    content: "实现安全模型：SD 存储/继承、GetSecurity/SetSecurity/GetSecurityByName"
    status: pending
  - id: kernel-cache
    content: "实现内核缓存策略 + FspFileSystemNotify 缓存失效"
    status: pending
  - id: unit-tests
    content: "单元测试：三阶段/TOCTOU/并发 Rename/SD 继承/缓存一致性"
    status: pending
isProject: false
---

# 子方案 ② - 文件系统与 WinFsp 集成

> 父方案：[VMem RAM Disk Design](./vmem_ram_disk_design_031231c8.plan.md)
> 对应代码：`src/VMem.Core/FileSystem/` + `src/VMem.Core/Memory/PagedFileContent.cs`
> Phase：1

---

## 1. FileNode - 文件/目录节点

```csharp
public sealed class FileNode : IDisposable
{
    // Microsoft 审查：构造函数确保不变量
    public static FileNode CreateFile(string name, PagePool pool, byte[]? securityDescriptor)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        return new FileNode(name, isDirectory: false, pool, securityDescriptor);
    }

    public static FileNode CreateDirectory(string name, byte[]? securityDescriptor)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        return new FileNode(name, isDirectory: true, pool: null, securityDescriptor);
    }

    private FileNode(string name, bool isDirectory, PagePool? pool, byte[]? sd)
    {
        Name = name;
        IsDirectory = isDirectory;
        Content = isDirectory ? null : new PagedFileContent(pool!);
        Children = isDirectory
            ? new ConcurrentDictionary<string, FileNode>(StringComparer.OrdinalIgnoreCase)
            : null;
        _securityDescriptor = sd?.ToArray(); // 深拷贝
        var now = DateTime.UtcNow.ToFileTimeUtc();
        CreationTime = LastAccessTime = LastWriteTime = ChangeTime = now;
        Attributes = isDirectory ? FileAttributes.Directory : FileAttributes.Archive;
    }

    public string Name { get; set; }
    public bool IsDirectory { get; }
    public FileAttributes Attributes { get; set; }

    // 时间戳（FILETIME 精度，100ns 刻度）
    public long CreationTime { get; set; }
    public long LastAccessTime { get; set; }
    public long LastWriteTime { get; set; }
    public long ChangeTime { get; set; }

    // 安全描述符（self-relative 二进制）
    // Oracle 审查：Get/Set 均深拷贝，防止外部持有引用修改内部状态
    private byte[]? _securityDescriptor;
    public byte[]? SecurityDescriptor
    {
        get => _securityDescriptor?.ToArray();
        set => _securityDescriptor = value?.ToArray();
    }

    // 文件/目录规则（辩论裁决 R19-1）：
    //   目录：Content=null, Children!=null
    //   文件（含 0 字节）：Content!=null, Children=null
    public PagedFileContent? Content { get; }
    public ConcurrentDictionary<string, FileNode>? Children { get; }
    // Children 构造时必须传入 StringComparer.OrdinalIgnoreCase（辩论裁决 R19-3/R20 P0）

    private int _openCount;
    public int IncrementOpen() => Interlocked.Increment(ref _openCount);
    public int DecrementOpen() => Interlocked.Decrement(ref _openCount);

    public bool PendingDelete { get; set; }

    // AllocationSize 维护（辩论裁决 R7）：
    // 语义 = max(物理已分配字节, 预留配额字节, FileSize)
    // GetFileInfo 直接读字段，不做 O(n) 扫描
    // 不变量：AllocationSize >= FileSize（Release Contract O(1)）
    public long AllocationSize => Content?.AllocationSize ?? 0;

    public void Dispose();
}
```

---

## 2. DirectoryTree - 线程安全目录树

### 2.1 锁层级定义（辩论裁决 R4/R7）

> **核心规则**：数字越小越先获取，**禁止逆序持有**。

```
L0 — PagePool 内部（ConcurrentStack / CAS，无阻塞锁，回调内可安全调用）
L1 — DirectoryTree 父目录锁（Rename 时字典序：先 lock 路径较小的父目录）
L2 — PagedFileContent._rwLock（per-file，RWLS）
L3 — FileNode 元数据锁（仅保护 RefCount/PendingDelete，持锁时间 < 1μs）

禁止：
- 持 L2 写锁时调用 L1（Rename 路径）
- 持 L1 时等待 L2 写锁（应：L1 内只做树操作，释放后再 I/O）

Cleanup/Close 锁序（R61 细化）：
- Cleanup: if PendingDelete → L1(Remove from parent) → release L1 → Notify（锁外）
- Close:   L3(DecrementOpen) → if RefCount==0 && PendingDelete → L2(Content.Dispose) → release L2
- 关键：L1 和 L2 不在同一线程同时持有；Dispose 在 Close 而非 Cleanup

FspNotify 必须在 L1/L2 全部释放后调用（WinFsp 可能重入）
```

### 2.2 API 设计

```csharp
public sealed class DirectoryTree
{
    private readonly FileNode _root;
    private readonly StringComparison _comparison = StringComparison.OrdinalIgnoreCase;
    private readonly ConcurrentDictionary<string, Lock> _dirLocks = new();

    public FileNode? Lookup(ReadOnlySpan<char> path);
    public NtStatus TryCreate(ReadOnlySpan<char> parentPath, FileNode node, out FileNode? existing);
    public NtStatus Rename(ReadOnlySpan<char> oldPath, ReadOnlySpan<char> newPath, bool replaceIfExists);
    public NtStatus TryRemove(ReadOnlySpan<char> path);
    public IReadOnlyList<FileNode> EnumerateDirectory(FileNode dir, string? marker);
}
```

**并发策略（辩论裁决 R7：Hybrid 方案）**：
- 每个目录的 `Children` = `ConcurrentDictionary<string, FileNode>`
- 单节点操作（Create/Delete/Lookup）→ `ConcurrentDictionary` 原子方法，**零树级锁**
- 跨目录 Rename → **字典序 per-parent `Lock`** 双锁（禁止全局 RWLS 和无锁 CAS）
- 同目录 Rename（大小写变更）→ 双锁退化单锁，`lockOld == lockNew` 不重复 Enter
- `EnumerateDirectory` → `Children.Values.ToArray()` 快照，枚举期间不持锁
- Rename Replace 语义：锁内 delete target → insert source，target Content 延迟到 Close Dispose

```csharp
public NtStatus Rename(ReadOnlySpan<char> oldPath, ReadOnlySpan<char> newPath, bool replace)
{
    ResolveParents(oldPath, newPath, out var oldParent, out var newParent, out var name, out var newName);
    var lockOld = _dirLocks.GetOrAdd(oldParent.CanonicalPath, _ => new Lock());
    var lockNew = _dirLocks.GetOrAdd(newParent.CanonicalPath, _ => new Lock());

    using var _ = AcquireOrdered(oldParent.Path, newParent.Path, lockOld, lockNew);

    // Microsoft 审查：先检查再 Remove，避免 rollback 失败导致节点丢失
    if (!oldParent.Children!.ContainsKey(name))
        return NtStatus.STATUS_OBJECT_NAME_NOT_FOUND;
    if (newParent.Children!.ContainsKey(newName) && !replace)
        return NtStatus.STATUS_OBJECT_NAME_COLLISION;
    // 校验通过后再执行 Remove（锁内保证无竞争）
    if (!oldParent.Children.TryRemove(name, out var node))
        return NtStatus.STATUS_OBJECT_NAME_NOT_FOUND; // 理论不可达，防御编程
    if (replace && newParent.Children.TryGetValue(newName, out var existing))
    {
        if (existing.IsDirectory && existing.Children!.Count > 0)
            return NtStatus.STATUS_DIRECTORY_NOT_EMPTY;
        // R51: existing.OpenCount>0 不阻止替换（NTFS 语义）
        // PendingDelete=true → Content Dispose 延迟到 Close(RefCount==0)
        existing.PendingDelete = true;
        newParent.Children.TryRemove(newName, out _);
    }
    node.Name = newName;
    newParent.Children[newName] = node;
    return NtStatus.STATUS_SUCCESS;
    // Notify 在此方法返回后、锁释放后调用
}
```

---

## 3. PagedFileContent - 三阶段写入协议

```csharp
public sealed class PagedFileContent : IDisposable
{
    private nint[] _pageTable;         // 0 = 稀疏未分配；动态倍增扩容（R32）
    // 页表扩容：写锁内 Array.Resize 倍增（旧表内容拷贝 + 新槽位=0）
    // 初始大小 = Math.Max(16, initialCapacity/pageSize)；最大 = int.MaxValue
    // V1 不缩容（Truncate 只清零页表项，不缩小数组）
    private readonly PagePool _pool;
    // 性能说明：ReaderWriterLockSlim 在高读并发下优于 Monitor，
    // 但在 .NET 9 中 Lock (System.Threading.Lock) 性能也不错。
    // 选择 RWLS 是因为 Read 操作远多于 Write（缓存命中时全走内核缓存不经过此处），
    // 三阶段写入 Phase 1 只需读锁，写锁仅在 Phase 3 持有极短时间（memcpy）。
    // 如果性能测试发现 RWLS 成为瓶颈，可替换为分段锁（按页索引范围分片）。
    private readonly ReaderWriterLockSlim _rwLock = new(LockRecursionPolicy.NoRecursion);
    private long _fileSize;
    private long _reservedPages;

    public long FileSize => Volatile.Read(ref _fileSize);
    private long _allocationSize;  // 增量维护，O(1) 读取
    public long AllocationSize => Volatile.Read(ref _allocationSize);

    public int Read(long offset, Span<byte> buffer)
    {
        _rwLock.EnterReadLock();
        try
        {
            // Clamp 到 FileSize（读锁内保证 _fileSize 不被并发截断）
            long fileSize = Volatile.Read(ref _fileSize);
            if (offset >= fileSize) return 0;
            int bytesToRead = (int)Math.Min(buffer.Length, fileSize - offset);
            int totalRead = 0;

            while (totalRead < bytesToRead)
            {
                int pageIndex = (int)((offset + totalRead) / _pool.PageSize);
                int pageOffset = (int)((offset + totalRead) % _pool.PageSize);
                int chunkSize = Math.Min(_pool.PageSize - pageOffset, bytesToRead - totalRead);

                nint page = pageIndex < _pageTable.Length ? _pageTable[pageIndex] : 0;
                if (page != 0)
                {
                    // 已分配页：memcpy
                    new Span<byte>((byte*)page + pageOffset, chunkSize)
                        .CopyTo(buffer.Slice(totalRead, chunkSize));
                }
                else
                {
                    // 稀疏页（Oracle 审查）：零填充而非跳过
                    buffer.Slice(totalRead, chunkSize).Clear();
                }
                totalRead += chunkSize;
            }
            return totalRead;
        }
        finally { _rwLock.ExitReadLock(); }
    }

    public NtStatus Write(long offset, ReadOnlySpan<byte> data, out int bytesWritten)
    {
        bytesWritten = 0;
        if (data.Length == 0) return NtStatus.STATUS_SUCCESS;

        // === Phase 1: 读锁扫描缺失页 ===
        // Google 审查：用 stackalloc/ArrayPool 替代 new List 减少 GC 压力
        int startPage = (int)(offset / _pool.PageSize);
        int endPage = (int)((offset + data.Length - 1) / _pool.PageSize);
        int spanCount = endPage - startPage + 1;
        int[]? rentedArr = spanCount > 64 ? ArrayPool<int>.Shared.Rent(spanCount) : null;
        Span<int> missingBuf = rentedArr != null ? rentedArr.AsSpan(0, spanCount) : stackalloc int[spanCount];
        int missingCount = 0;

        _rwLock.EnterReadLock();
        try
        {
            EnsurePageTableCapacity(endPage + 1);
            for (int i = startPage; i <= endPage; i++)
                if (_pageTable[i] == 0) missingBuf[missingCount++] = i;
        }
        finally { _rwLock.ExitReadLock(); }
        var missingIndices = missingBuf[..missingCount];

        // === Phase 2: 锁外批量分配（BatchLease 保护异常路径）===
        using var batch = _pool.RentBatch(missingIndices.Count);
        if (batch.Count < missingIndices.Count)
            return NtStatus.STATUS_DISK_FULL;

        // === Phase 3: 写锁赋值 + memcpy ===
        _rwLock.EnterWriteLock();
        try
        {
            int batchIdx = 0;
            for (int i = 0; i < missingIndices.Count; i++)
            {
                int pageIdx = missingIndices[i];
                if (_pageTable[pageIdx] == 0) // TOCTOU 重检
                    _pageTable[pageIdx] = batch.Commit(batchIdx);
                batchIdx++;
            }

            // 逐页 memcpy（Google 审查：禁止跨页 CopyBlock）
            int written = 0;
            while (written < data.Length)
            {
                int pageIdx = (int)((offset + written) / _pool.PageSize);
                int pageOff = (int)((offset + written) % _pool.PageSize);
                int chunk = Math.Min(_pool.PageSize - pageOff, data.Length - written);
                data.Slice(written, chunk).CopyTo(
                    new Span<byte>((byte*)_pageTable[pageIdx] + pageOff, chunk));
                written += chunk;
            }

            // 事务提交：FileSize + AllocationSize 原子更新
            long newEnd = offset + data.Length;
            if (newEnd > _fileSize) Volatile.Write(ref _fileSize, newEnd);
            _allocationSize = Math.Max(_allocationSize, CeilToPageBoundary(newEnd));
            bytesWritten = data.Length;
        }
        finally { _rwLock.ExitWriteLock(); }

        if (rentedArr != null) ArrayPool<int>.Shared.Return(rentedArr);
        return NtStatus.STATUS_SUCCESS;
    }
    public NtStatus SetLength(long newLength)
    {
        if (newLength < 0) return NtStatus.STATUS_INVALID_PARAMETER;

        _rwLock.EnterWriteLock();
        try
        {
            long oldSize = _fileSize;
            if (newLength == oldSize) return NtStatus.STATUS_SUCCESS;

            if (newLength > oldSize)
            {
                // 扩展：预留配额（稀疏语义，不分配物理页）
                int oldPages = CeilDiv(oldSize, _pool.PageSize);
                int newPages = CeilDiv(newLength, _pool.PageSize);
                int additionalPages = newPages - oldPages;
                if (additionalPages > 0 && !_pool.TryReserve(additionalPages))
                    return NtStatus.STATUS_DISK_FULL;
                EnsurePageTableCapacity(newPages);
                _reservedPages += additionalPages;
            }
            else
            {
                // 截断：释放超出页 + 最后页部分清零 + 归还预留
                int keepPages = CeilDiv(newLength, _pool.PageSize);
                int oldPages = CeilDiv(oldSize, _pool.PageSize);

                // 释放 [keepPages, oldPages) 范围的物理页
                for (int i = keepPages; i < oldPages && i < _pageTable.Length; i++)
                {
                    if (_pageTable[i] != 0)
                    {
                        _pool.Return(_pageTable[i]);
                        _pageTable[i] = 0;
                        _allocationSize -= _pool.PageSize;
                    }
                }

                // 最后保留页内部分清零（Google 审查：防信息泄漏）
                if (keepPages > 0 && _pageTable[keepPages - 1] != 0)
                {
                    int tail = (int)(newLength % _pool.PageSize);
                    if (tail > 0)
                        NativeMemory.Clear((byte*)_pageTable[keepPages - 1] + tail,
                            (nuint)(_pool.PageSize - tail));
                }

                // 归还多余预留
                int releasedReservation = Math.Max(0,
                    (int)_reservedPages - keepPages + CeilDiv(oldSize, _pool.PageSize) - oldPages);
                if (releasedReservation > 0)
                {
                    _pool.Unreserve(releasedReservation);
                    _reservedPages -= releasedReservation;
                }
            }

            Volatile.Write(ref _fileSize, newLength);
            _allocationSize = Math.Max(_allocationSize, CeilToPageBoundary(newLength));
            return NtStatus.STATUS_SUCCESS;
        }
        finally { _rwLock.ExitWriteLock(); }
    }

    public void Dispose()
    {
        _rwLock.EnterWriteLock();
        try
        {
            for (int i = 0; i < _pageTable.Length; i++)
            {
                if (_pageTable[i] != 0)
                {
                    _pool.Return(_pageTable[i]);
                    _pageTable[i] = 0;
                }
            }
            if (_reservedPages > 0)
            {
                _pool.Unreserve(_reservedPages);
                _reservedPages = 0;
            }
        }
        finally { _rwLock.ExitWriteLock(); }
        _rwLock.Dispose();
    }
}
```

### 三阶段写入流程（学习自 hooyao/RamDrive）

```
Phase 1 — 读锁扫描
    ├─ EnterReadLock()
    ├─ 扫描 [offset, offset+len) 涉及的页索引
    ├─ 收集 _pageTable[i] == 0 的缺失页列表
    └─ ExitReadLock()

Phase 2 — 锁外无锁批量分配
    ├─ 不持有任何锁
    ├─ _pool.TryRentBatch() 或 TryRentFromReservation()
    ├─ 用 PageLease[] 保护，异常自动归还
    └─ 分配失败 → STATUS_DISK_FULL

Phase 3 — 写锁赋值 + memcpy
    ├─ EnterWriteLock()
    ├─ 再次检查 _pageTable[i] == 0（TOCTOU 防护）
    │   └─ 已被并发写入填充 → 归还多余分配
    ├─ 赋值 _pageTable[i] = allocatedPage
    ├─ NativeMemory.Copy 数据到页
    ├─ 更新 _fileSize
    └─ ExitWriteLock()
```

**核心优势**：写锁持有时间从 O(n × alloc + n × memcpy) **降为 O(n × memcpy)**。

**Phase 2~3 TOCTOU 处理**：
- Phase 3 写锁内再次检查每个页槽位是否仍为空
- 并发写入已填充的页 → 归还 Phase 2 分配的多余页
- CAS 调整 `_reservedPages` 避免计数漂移

### SetLength 语义（辩论裁决 R53-R55）

**扩展**（newSize > oldSize）：
1. 计算需要的新页数：`newPages = CeilDiv(newSize, pageSize) - CeilDiv(oldSize, pageSize)`
2. `_pool.TryReserve(newPages)` → 失败返回 STATUS_DISK_FULL
3. 写锁内更新 `_fileSize` 和 `_allocationSize`（仅增量，不分配物理页 → 稀疏语义）

**截断**（newSize < oldSize）：
1. 写锁内：
   - 计算释放页范围：`[CeilDiv(newSize, pageSize)..CeilDiv(oldSize, pageSize))`
   - 遍历释放范围，对每个 `_pageTable[i] != 0` 的页：`_pool.Return(page)` + `_pageTable[i] = 0`
   - 更新 `_fileSize = newSize`，`_allocationSize` 减少已释放的物理页 + Unreserve 多余预留
2. 如果截断到文件中间（非页对齐）：最后一个保留页内 `newSize % pageSize` 之后的字节需要清零
3. Unreserve：`releasedReservation = oldReservedPages - newNeededPages`，`_pool.Unreserve(releasedReservation)`

---

## 4. VMemFileSystem - WinFsp 回调

### 4.0 WinFsp API 陷阱（辩论裁决 R101-R110）

| 陷阱 | 说明 | 应对 |
|------|------|------|
| **异常逃逸** | 回调中未捕获异常 → WinFsp 内核崩溃/未定义行为 | 每个回调顶层 try-catch → 返回 STATUS_INTERNAL_ERROR + ERR 日志 |
| **路径格式** | 回调路径 = `\dir\file.txt`（反斜杠，相对根） | 不要用 `Path.Combine`；手动拼接 |
| **Mount 阻塞** | `StartDispatcher()` 阻塞直到 `StopDispatcher()` | CLI 在独立线程运行；Service 用 Task.Run |
| **Dispose 顺序** | StopDispatcher → 等回调完成 → Delete → Dispose 文件树 → Dispose PagePool | 严格顺序，否则 UAF |
| **FileContext** | WinFsp 透传 `object` FileContext 给所有后续回调 | 存 FileNode 引用；不要存路径（Rename 后失效） |
| **回调线程** | WinFsp 线程池回调；同一文件可能不同线程 | 所有 FileNode 操作必须线程安全 |
| **SectorSize 对齐** | Read/Write offset 和 length 已被 WinFsp 对齐到 SectorSize | 用户态无需再对齐 |

```csharp
public sealed class VMemFileSystem : IFileSystem
{
    private readonly DirectoryTree _tree;
    private readonly PagePool _pool;
    private readonly StatsCollector _stats;
    private readonly FileSystemHost _host;     // 用于 Notify

    // --- 卷信息（R21）---
    NtStatus GetVolumeInfo(out VolumeInfo info);
    // TotalSize = PagePool.TotalCapacityBytes
    // FreeSize  = TotalCapacityBytes - UsedBytes - ReservedBytes
    // VolumeLabel, MaxComponentLength = 255

    // --- 命名空间 ---
    NtStatus Create(...);      // 设置 CreationTime/LastAccessTime/LastWriteTime/ChangeTime
    NtStatus Open(...);
    NtStatus Overwrite(...);   // R52: SetLength(0) + 重置 Attributes + 更新 LWT/CT
                               // 保留 SD 和 CreationTime；FILE_ATTRIBUTE_ARCHIVE 置位
    NtStatus Rename(...);      // → Notify（锁外）

    // --- 生命周期 ---
    void Cleanup(...);         // PendingDelete + Notify（详见 Cleanup/Close 语义）
    void Close(...);           // RefCount-- → Dispose

    // --- 数据 ---
    NtStatus Read(...);        // 隐式更新 LastAccessTime（R24）
    NtStatus Write(...);       // 三阶段；隐式更新 LastWriteTime + ChangeTime
    NtStatus Flush(...);       // RAM 盘 = no-op，返回 STATUS_SUCCESS
    NtStatus SetFileSize(...); // SetLength + 预留

    // --- 目录枚举（R23/R56-R58）---
    NtStatus ReadDirectory(...);
    // 1. Marker == null → 从头开始；Marker != null → 续读
    // 2. "." 和 ".." 由 WinFsp 自动注入（SectorSize != 0 时）
    // 3. 实现伪码：
    //    var entries = dir.Children!.Values.ToArray();  // 快照
    //    Array.Sort(entries, (a,b) => string.Compare(a.Name, b.Name, OrdinalIgnoreCase));
    //    foreach (var e in entries)
    //        if (marker == null || string.Compare(e.Name, marker, OrdinalIgnoreCase) > 0)
    //            if (!FillEntry(e)) break;  // 缓冲区满
    //    return STATUS_SUCCESS;
    // 4. 注意：不返回 STATUS_NO_MORE_FILES；WinFsp 看到 FillEntry 未填满即知结束

    // --- 元数据 ---
    NtStatus GetFileInfo(...);  // AllocationSize = Content?.AllocationSize ?? 0
    NtStatus SetBasicInfo(...); // 时间戳：0 = 不更新，-1 (UINT64_MAX) = 不更新（R24）

    // --- 安全 ---
    NtStatus GetSecurity(...);       // 返回 SD 深拷贝
    NtStatus SetSecurity(...);       // 存储 SD 深拷贝
    NtStatus GetSecurityByName(...); // 路径 Walk 查 node/parent

    // --- 可选（Phase 2+）---
    // SetDelete, GetDirInfoByName, GetStreamInfo, GetReparsePoint, SetReparsePoint
}
```

**Cleanup vs Close 语义（辩论裁决 R4/R7：memfs 骨架 + C# 适配）**：

```
Open  → RefCount++
Cleanup（最后句柄关闭）
    → 若 FILE_DELETE_ON_CLOSE → PendingDelete = true
    → 若 PendingDelete → Notify(Delete)（锁外发送）
    → 树节点 Remove 延迟到 Close RefCount==0
    → 禁止在 Cleanup 同步 Dispose 页表
Close → RefCount--
    → 若 PendingDelete && RefCount == 0
        → Dispose 内容（释放页面 + 归还 PagePool）
    → 否则仅减引用计数
```

> **关键**：Cleanup 里不同步释放页——这是 RamDrive 早期 bug 来源。

---

## 5. WinFsp 内核缓存设计

### 5.1 缓存策略配置（辩论裁决 R4：Phase 1 默认 Safe）

| 模式 | FileInfoTimeout | 行为 | Phase 1 |
|------|-----------------|------|---------|
| **Safe（默认）** | 0 | 无缓存。最安全，winfsp-tests 主跑模式 | ✓ 主验收 |
| Balanced | 1000 | 元数据缓存 1 秒，无数据缓存 | 可选冒烟 |
| Fast | uint.MaxValue | 完整数据+元数据缓存 + Fast I/O。**最快** | ✓ 缓存一致性测试 |

CLI 参数：`vmem mount R: --size 1G --cache safe|balanced|fast`（默认 `safe`）。
Phase 1 验收完成后可改默认为 `balanced` 或 `fast`。

### 5.1.1 FileSystemHost 参数（R43）

```csharp
var host = new FileSystemHost(fs);
host.SectorSize = 4096;                    // 与 PageSize 对齐
host.SectorsPerAllocationUnit = 1;         // 分配单元 = 1 扇区
host.MaxComponentLength = 255;             // NTFS 兼容
host.FileInfoTimeout = cacheMode switch {
    "safe" => 0,
    "balanced" => 1000,
    "fast" => uint.MaxValue,
    _ => 0
};
host.CaseSensitiveSearch = false;          // Windows 标准
host.UnicodeOnDisk = true;
host.PersistentAcls = true;               // 启用 SD 支持
host.ReparsePoints = false;               // V1 不支持
host.NamedStreams = false;                 // V1 不支持
host.VolumeSerialNumber = 0;              // 随机生成
host.FlushAndPurgeOnCleanup = false;      // RAM 盘无需 flush

host.Mount(driveLetter);                  // 阻塞直到 StopDispatcher
```

### 5.2 缓存失效通知（FspFileSystemNotify）

当 `EnableKernelCache=true` 时，以下操作**必须**在完成后通知内核：

| 操作 | 通知参数 | 说明 |
|------|----------|------|
| `Rename` | 旧路径 + 新路径 | 更新缓存键 |
| `Delete`（Cleanup） | 被删路径 | 清除缓存条目 |
| `Overwrite` | 文件路径 | 刷新大小/属性 |
| `SetFileSize` | 文件路径 | 更新 FileSize/AllocationSize |

**实现要求**（辩论裁决 R4/R7）：
- **时机**：锁内变异 → 解锁 → **同步** Notify → return SUCCESS。**禁止 defer/异步队列**
- 零托管堆分配：`stackalloc char[260]` 构建 UTF-16 路径缓冲区
- 路径格式：`\Dir\File.txt`
- `EnableKernelCache=false` 时 Notify 是 no-op（`if (!_enableKernelCache) return;` 守卫）
- SetBasicInfo（mtime 变更）和 SetFileSize 也要 Notify
- Rename 发两次：先 `REMOVED(oldPath)` 后 `ADDED(newPath)`
- 开销：每次变更多一次 P/Invoke + 小量 memcpy

```csharp
private void NotifyIfEnabled(ReadOnlySpan<char> path, uint action)
{
    if (!_enableKernelCache) return;
    Span<char> buf = stackalloc char[260];
    BuildNtPath(path, buf, out int len);
    _host.Notify(buf[..len], action);
}
```

---

## 6. 安全模型

### 6.1 设计原则

1. 每个 FileNode 持有 **self-relative SecurityDescriptor**（`byte[]`）
2. `Create` 回调接收 WinFsp 传入的 SD（内核已计算好继承） → 直接存储
3. `GetSecurityByName` 在 Open 前被调用 → 必须正确返回
4. **WinFsp 内核侧执行 ACL 检查**，用户态只存储/返回

### 6.2 默认 Root SDDL

> ⚠ **安全审查意见**：原方案默认 `Everyone:FA` 过于宽松。在 Service 以 LocalSystem 运行的场景下，
> 任何低权限进程都可以读写 RAM 盘上的所有文件，存在数据泄漏和越权写入风险。

**分场景默认 SDDL**：

| 场景预设 | 默认 SDDL | 说明 |
|---------|----------|------|
| 浏览器缓存 / TEMP | `D:P(A;;FA;;;BA)(A;;FA;;;SY)(A;;FA;;;IU)` | **仅当前交互用户 (IU)** + Admin + SYSTEM |
| 开发编译 | `D:P(A;;FA;;;BA)(A;;FA;;;SY)(A;;FA;;;IU)` | 同上 |
| 自定义（高级） | 用户自填 SDDL | 完全控制 |
| 传统兼容 | `D:P(A;;FA;;;BA)(A;;FA;;;SY)(A;;FA;;;WD)` | Everyone:FA（需勾选"允许所有用户访问"） |

**GUI 创建对话框联动**：选择场景预设时自动应用对应 SDDL，"高级选项"中可覆盖。
自定义模式默认 `IU`（交互用户），如果用户勾选"允许所有用户访问"则切换为 `WD`（Everyone）。

支持通过配置自定义 Root SDDL（学习自 memefs `-S` 参数）。

---

## 7. 日志与可观测性（L3 范式补丁）

> 详见 [子方案 ⑦ - 日志与可观测性](./vmem_07_logging_observability.plan.md)

### 7.1 WinFsp 回调 CorrelationId 追踪

每个 WinFsp 回调入口生成 `CorrelationId`，贯穿整条调用链：

```csharp
public NtStatus Write(/* ... */)
{
    using var _ = OperationContext.BeginOperation("Write");
    // correlationId 自动附加到此后所有日志
    // ...
    _logger.Information("FS.Write OK path={Path} offset={Offset} bytes={Bytes} totalMs={Ms}",
        path, offset, bytesWritten, sw.Elapsed.TotalMilliseconds);
    return NtStatus.STATUS_SUCCESS;
}
```

### 7.2 热路径日志零分配策略

> **性能审查意见**：每次 FS 回调都做结构化日志会产生字符串格式化和装箱分配。
> 在 `EnableKernelCache=false`（每次 I/O 都经过回调）的场景下，日志开销不可忽略。

**原则**：
- `VRB`/`DBG` 级别日志使用 `_logger.IsEnabled(LogEventLevel.Debug)` **前置检查**，未启用时零开销
- `INF` 级别出口摘要日志使用 Serilog **消息模板**（编译时生成，避免字符串插值分配）
- 禁止在热路径使用 `$""` 插值字符串，统一使用 `{PropertyName}` 模板占位符
- `CorrelationId` 通过 `Serilog.Context.LogContext.PushProperty` 注入，避免每条日志手动传参

```csharp
// 正确 ✓：消息模板，零分配直到实际写入
_logger.Information("FS.Write OK path={Path} bytes={Bytes}", path, bytesWritten);

// 错误 ✗：字符串插值，每次都分配
_logger.Information($"FS.Write OK path={path} bytes={bytesWritten}");
```

### 7.3 全部回调日志规范

| 回调 | 入口日志 (DBG) | 出口日志 (INF) | 失败日志 (ERR) |
|------|---------------|---------------|---------------|
| Create | 路径, 模式, 安全描述符长度 | 路径, FileNode 类型, AllocationSize | NtStatus, 失败原因 |
| Open | 路径 | 路径, OpenCount | NtStatus |
| Write | 路径, offset, 请求字节数 | 路径, 写入字节数, 分配页数, 三阶段耗时 | NtStatus |
| Read | 路径, offset, 请求字节数 | 路径, 实际读取字节数, 耗时 | NtStatus |
| Rename | 旧路径, 新路径, replaceIfExists | 旧路径→新路径, 耗时, Notify已发送 | NtStatus |
| Cleanup | 路径, PendingDelete? | 路径, 是否执行删除, Notify已发送 | - |
| Close | 路径, 最终OpenCount | 路径, 资源释放详情 | - |
| SetFileSize | 路径, 旧大小, 新大小 | 路径, 预留/释放页数 | NtStatus |

### 7.3 三阶段写入分阶段计时

```csharp
public NtStatus Write(long offset, ReadOnlySpan<byte> data, out int bytesWritten)
{
    var swTotal = Stopwatch.StartNew();

    // Phase 1
    var sw1 = Stopwatch.StartNew();
    _rwLock.EnterReadLock();
    // ... scan missing pages ...
    _rwLock.ExitReadLock();
    var phase1Ms = sw1.Elapsed.TotalMilliseconds;

    // Phase 2
    var sw2 = Stopwatch.StartNew();
    // ... allocate pages (no lock) ...
    var phase2Ms = sw2.Elapsed.TotalMilliseconds;

    // Phase 3
    var sw3 = Stopwatch.StartNew();
    _rwLock.EnterWriteLock();
    // ... assign + memcpy ...
    _rwLock.ExitWriteLock();
    var phase3Ms = sw3.Elapsed.TotalMilliseconds;

    var totalMs = swTotal.Elapsed.TotalMilliseconds;
    if (totalMs > 10.0)
    {
        _logger.Warning("SLOW OPERATION Write path={Path} totalMs={TotalMs} " +
            "phase1={P1}ms phase2={P2}ms phase3={P3}ms",
            path, totalMs, phase1Ms, phase2Ms, phase3Ms);
    }
}
```

### 7.4 FspNotify 调用日志

每次缓存失效通知均记录，便于 AI 排查缓存一致性问题：

```csharp
_logger.Debug("FspNotify {Action} path={Path}", "Rename", oldPath);
_logger.Debug("FspNotify {Action} path={Path}", "Delete", deletedPath);
```

### 7.5 DirectoryTree 操作审计

| 操作　　　　　　　　　 | 日志级别 | 记录内容　　　　　　　　　　　　　　　　 |
| ------------------------| ----------| ------------------------------------------|
| TryCreate 成功　　　　 | `INF`　　| 父路径, 节点名, 是否目录　　　　　　　　 |
| TryCreate 冲突　　　　 | `DBG`　　| 父路径, 冲突节点名　　　　　　　　　　　 |
| Rename 成功　　　　　　| `INF`　　| 旧路径→新路径, 是否覆盖, 加锁等待耗时　　|
| Rename 加锁等待 > 50ms | `WRN`　　| 旧路径, 新路径, 等待耗时（可能死锁预兆） |
| TryRemove 成功　　　　 | `INF`　　| 路径　　　　　　　　　　　　　　　　　　 |
| TryRemove 失败（非空） | `DBG`　　| 路径, Children 数量　　　　　　　　　　　|

### 7.6 契约检查（Release 生效）

| 契约名 | 检查时机 | 表达式 |
|--------|---------|--------|
| `PageTableIntegrity` | Write Phase 3 完成后 | 所有非零页表项指向有效 NativeMemory 地址 |
| `FileSizeConsistent` | SetLength 完成后 | `fileSize >= 0 && fileSize <= maxPages * pageSize` |
| `NotifyAfterMutation` | Rename/Delete/Overwrite 后 | `FspNotify` 已调用（EnableKernelCache 时） |
| `NoOrphanNodes` | TryRemove 完成后 | 被删节点不在任何父目录 Children 中 |
| `OpenCountNonNegative` | DecrementOpen 后 | `_openCount >= 0` |

---

## 8. 边界场景（辩论裁决 R19）

| 场景 | 处理策略 |
|------|---------|
| **0 字节文件** | `PagedFileContent` 非 null 空实例（`Content!=null, _pageTable` 长度=0）；不能用 `Content==null` 判断"无内容" |
| **超长路径** | V1 限制组件名 ≤255 字符、全路径 ≤1024 字符；`Create/Rename` 入口校验，`Notify stackalloc char[260]` 须 guard |
| **大小写 Rename** | 同目录改名 `FOO→foo` 需 `ConcurrentDictionary(OrdinalIgnoreCase)` 保证不冲突 |
| **创建-删除-创建** | Delete-on-close 名称占用窗口是设计（非 bug）；`PendingDelete=true` 时 `Create` 同名返回冲突 |
| **4GB 单文件** | `long` 页索引 + `int[]` 页表 + `EnsurePageTableCapacity` 倍增；V1 最大 = PagePool 容量 |
| **kill -9** | OS 回收 NativeMemory；数据全丢符合 RAM 盘语义；无需 finalizer 兜底 |
| **Read 超出 FileSize** | `Math.Min(bytesToRead, FileSize - offset)` clamp；不返回未初始化内存 |
