# Bump Allocator

本练习位于 `exercises/02_no_std_dev/02_bump_allocator/src/lib.rs`，实现一种简单的 bump allocator（线性分配器）。它只维护一个不断向前移动的指针，用于从固定堆区中依次分配内存。

## 1. 基本结构

`BumpAllocator` 保存三个地址：

```rust
pub struct BumpAllocator {
    heap_start: usize,
    heap_end: usize,
    next: AtomicUsize,
}
```

- `heap_start`：堆区起始地址。
- `heap_end`：堆区结束地址，不包含该地址。
- `next`：下一次分配的候选地址。

初始状态下，`next` 等于 `heap_start`。每次分配成功后，`next` 向高地址移动。

```text
heap_start                         heap_end
|--- 已分配区域 ---| next |--- 可用区域 ---|
```

## 2. 内存对齐

不同类型对地址有不同的对齐要求。例如某些类型必须从 8 字节边界开始。分配前需要将 `next` 向上调整到满足 `layout.align()` 的地址：

```rust
let aligned = (addr + align - 1) & !(align - 1);
```

该公式要求 `align` 是 2 的幂，这正是 `Layout` 的对齐约束。

例如：

- `addr = 100`
- `align = 8`
- 对齐后地址为 `104`

分配返回的指针必须满足：

```rust
ptr as usize % layout.align() == 0
```

## 3. 检查空间是否足够

对齐后需要计算本次分配的结束地址：

```rust
let end = aligned + layout.size();
```

如果 `end > heap_end`，说明堆区空间不足，`alloc` 应返回空指针：

```rust
return null_mut();
```

调用者需要把空指针视为分配失败，不能继续解引用它。

实际的底层实现还应注意地址加法溢出问题；如果 `aligned + layout.size()` 溢出，也必须视为分配失败。

## 4. 使用 CAS 保证并发安全

`GlobalAlloc::alloc` 可能被多个线程同时调用，因此不能只读取和写入普通变量。这里使用 `AtomicUsize` 和 `compare_exchange`：

```rust
let current = self.next.load(Ordering::SeqCst);
// 计算 aligned 和 end
self.next.compare_exchange(
    current,
    end,
    Ordering::SeqCst,
    Ordering::SeqCst,
)?;
```

CAS（compare-and-swap）的含义是：只有当 `next` 仍然等于读取到的 `current` 时，才把它更新为 `end`。

如果 CAS 失败，说明另一个线程已经抢先分配了空间。此时必须重新读取 `next`、重新计算对齐地址和结束地址，然后继续尝试。通常实现为循环：

```text
读取 next
计算 aligned 和 end
检查是否越界
CAS 更新 next
成功则返回地址，失败则重试
```

只使用一次 CAS 而不重试，可能返回与实际原子状态不一致的地址，导致多个分配发生重叠。

## 5. `AtomicUsize` 详解

`AtomicUsize` 是一个可以被多个线程安全访问的原子 `usize`。它的大小与当前目标平台的指针大小一致，因此适合保存地址整数：

```rust
let next = AtomicUsize::new(heap_start);
```

普通的 `usize` 更新不是一个不可分割的操作。例如：

```text
线程 A 读取 next = 100
线程 B 读取 next = 100
线程 A 写入 next = 120
线程 B 写入 next = 136
```

如果两个线程都根据旧值 `100` 计算地址，它们可能返回重叠的内存区域。`AtomicUsize` 允许读取、写入和比较更新在硬件层面作为原子操作完成，避免读写过程被其他线程插入。

常用操作包括：

```rust
// 原子读取
let current = self.next.load(Ordering::SeqCst);

// 原子写入
self.next.store(self.heap_start, Ordering::SeqCst);

// 当 current 仍然是旧值时，原子地更新为 new
let result = self.next.compare_exchange(
    current,
    new,
    Ordering::SeqCst,
    Ordering::SeqCst,
);
```

`compare_exchange` 返回：

- `Ok(current)`：比较成功，当前线程成功占用 `[aligned, end)` 区域。
- `Err(actual)`：比较失败，说明其他线程已经修改了 `next`；`actual` 是最新值，需要重新计算并重试。

因此 CAS 分配循环的关键不是“读取后直接写入”，而是“只有确认读取值没有变化时才提交更新”：

```rust
loop {
    let current = self.next.load(Ordering::SeqCst);
    let aligned = align_up(current, layout.align());
    let end = aligned + layout.size();

    if end > self.heap_end {
        return null_mut();
    }

    match self.next.compare_exchange(
        current,
        end,
        Ordering::SeqCst,
        Ordering::SeqCst,
    ) {
        Ok(_) => return aligned as *mut u8,
        Err(_) => continue,
    }
}
```

### 内存序 `Ordering`

原子操作除了保证单次操作不可分割，还需要指定内存序。`Ordering::SeqCst`（顺序一致性）提供最直观的保证：所有线程观察到的原子操作顺序保持一致，适合本练习这种需要清晰理解并发行为的实现。

CAS 的成功和失败路径分别接收一个内存序参数。失败路径不能使用 `Release` 或 `AcqRel`，通常使用 `Relaxed` 或本练习中的 `SeqCst`。在更复杂的无锁结构中，可以根据实际同步关系选择更弱的内存序以提升性能，但必须证明其正确性。

`AtomicUsize` 只保护 `next` 这个分配游标；它不会自动初始化或保护堆区中的对象内容。返回指针后，调用者仍需遵守分配大小、对齐和生命周期要求。

## 6. 为什么 `dealloc` 为空

Bump allocator 只会向前移动 `next`，不维护空闲链表，因此无法单独回收中间的一块内存：

```rust
unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
    // 不回收单个对象
}
```

这意味着已分配内存会一直占用空间，直到整个 allocator 被重置。它适合：

- 生命周期相近、批量释放的对象。
- 内核早期启动阶段。
- 临时 arena 或线性内存区域。

不适合频繁分配和释放不同生命周期对象的场景。

## 7. `reset` 的作用

```rust
pub fn reset(&self) {
    self.next.store(self.heap_start, Ordering::SeqCst);
}
```

`reset` 将分配指针恢复到堆区起点，相当于一次性释放所有之前的分配。调用者必须确保旧对象不再使用，否则会产生悬空指针和未定义行为。

## 8. `unsafe` 安全要求

`BumpAllocator::new` 和 `GlobalAlloc` 操作都涉及裸地址，因此调用者必须保证：

1. `heap_start..heap_end` 是有效、可写的内存区域。
2. 区间没有被其他 allocator 或代码同时使用。
3. `heap_start <= heap_end`，且地址计算不会溢出。
4. 分配返回的地址满足请求的大小和对齐要求。
5. `reset` 时不存在仍在使用的旧分配。

Rust 的类型系统无法自动验证这些裸指针约束，因此必须由 allocator 的实现者和调用者共同维护。

## 9. 核心要点

1. Bump allocator 通过移动一个指针完成分配，算法简单且速度快。
2. 每次分配都必须先处理对齐，再检查结束地址是否越界。
3. 并发分配需要使用 `AtomicUsize` 和 CAS 循环。
4. `dealloc` 通常不回收单个对象，空间在 `reset` 时统一释放。
5. 返回空指针表示堆空间不足。
6. 裸指针和固定堆区带来 `unsafe` 要求，必须严格维护内存边界和生命周期。
