# 空闲链表分配器

本练习位于 `exercises/02_no_std_dev/03_free_list_allocator/src/lib.rs`，在 bump allocator 的基础上增加释放和内存复用能力。

## 1. 基本思想

Bump allocator 只会向前移动分配指针，释放的内存无法再次使用。空闲链表分配器则把已经释放的区域串成链表：

```text
free_list -> [block A] -> [block B] -> [block C] -> null
```

每个空闲块的开头保存一个 `FreeBlock` 头部：

```rust
struct FreeBlock {
    size: usize,
    next: *mut FreeBlock,
}
```

链表节点直接存放在空闲内存本身，因此称为 intrusive linked list（侵入式链表）。不需要额外分配节点空间。

## 2. First-fit 分配策略

分配时首先从 `free_list` 头部开始遍历：

```text
当前节点 -> 下一个节点 -> 下一个节点 -> ...
```

找到第一个同时满足以下条件的块即可复用：

1. 块大小不小于本次请求的大小。
2. 块起始地址满足请求的对齐要求。

这种策略叫 first-fit（首次适配）。找到节点后，需要把它从链表中摘除：

- 如果节点是头节点，更新 `free_list`。
- 如果节点位于中间，更新前一个节点的 `next`。

## 3. 找不到空闲块时使用 bump 区域

如果空闲链表中没有合适的块，分配器从 `bump_next` 指向的未使用区域分配：

```text
heap_start                  bump_next                 heap_end
|------ 已使用区域 ---------|-------- 可用区域 ---------|
```

这部分逻辑与 bump allocator 类似：

1. 将 `bump_next` 向上对齐。
2. 计算 `aligned + size`。
3. 检查是否超过 `heap_end`。
4. 使用 CAS 更新 `bump_next`。

如果空间不足，则返回 `null_mut()`。

## 4. 释放内存

释放时，分配器把指针转换为 `*mut FreeBlock`，并在这块内存的起始位置写入链表头部：

```rust
let free_block = ptr as *mut FreeBlock;
free_block.write(FreeBlock {
    size,
    next: head,
});
self.set_free_list_head(free_block);
```

释放块采用头插法加入链表，因此操作复杂度为 `O(1)`。下次分配会优先检查这个新加入的块。

由于头部本身也要存储 `FreeBlock`，释放时记录的大小至少应为：

```rust
layout.size().max(core::mem::size_of::<FreeBlock>())
```

否则未来复用该块时可能没有空间保存链表头部。

## 5. `*mut T` 指针操作

空闲链表使用裸指针连接节点：

```rust
let mut curr: *mut FreeBlock = self.free_list_head();
while !curr.is_null() {
    let block = &*curr;
    curr = block.next;
}
```

常见操作包括：

- `curr.is_null()`：判断是否到达链表末尾。
- `&*curr`：将有效裸指针临时视为引用。
- `ptr.write(value)`：向指定地址写入值。
- `ptr as *mut u8`：将节点指针转换为分配结果。

这些操作都是 `unsafe`，调用者必须保证指针非空、对齐正确，并且指向仍然属于 allocator 管理的有效内存区域。

## 6. 对齐与块大小

分配器将请求的大小和对齐分别与 `FreeBlock` 的要求取最大值：

```rust
let size = layout.size().max(core::mem::size_of::<FreeBlock>());
let align = layout.align().max(core::mem::align_of::<FreeBlock>());
```

这样可以确保：

- 返回的空间足以容纳未来的 `FreeBlock` 头部。
- 空闲块地址满足头部结构体的对齐要求。
- 返回地址仍然满足请求的对齐要求。

如果空闲块地址不满足更高的对齐要求，即使空间足够，也不能直接复用该块。

## 7. 链表与并发保护

测试环境使用 `std::sync::Mutex` 保护链表头；`no_std` 环境使用 `UnsafeCell` 保存链表指针：

```rust
#[cfg(test)]
free_list: std::sync::Mutex<*mut FreeBlock>,

#[cfg(not(test))]
free_list: core::cell::UnsafeCell<*mut FreeBlock>,
```

`UnsafeCell` 只允许内部可变访问，不会自动提供线程同步。若 allocator 需要支持多个线程同时分配和释放，还需要额外的锁或无锁链表设计。

`bump_next` 使用 `AtomicUsize` 和 CAS 保护 bump 区域的分配游标；CAS 失败时必须重新读取最新值并重试。

## 8. 内存碎片

空闲链表可以复用释放的内存，但会产生外部碎片：

```text
[空闲 32B] [已使用] [空闲 64B] [已使用] [空闲 32B]
```

即使空闲总量足够，也可能没有一块连续空间满足较大的请求。基础实现通常不合并相邻空闲块，也不进行分割；更完善的 allocator 可以增加：

- 相邻块合并（coalescing）。
- 大块分割（splitting）。
- 更高效的分区或大小分类。

## 9. 核心要点

1. 空闲链表把释放的内存块组织成侵入式链表。
2. first-fit 会选择第一个满足大小和对齐要求的块。
3. 没有合适空闲块时，分配器回退到 bump 区域。
4. `dealloc` 使用头插法，释放操作为 `O(1)`。
5. 裸指针操作必须满足有效性、对齐和生命周期要求。
6. `AtomicUsize` 保护 bump 游标，链表本身还需要独立的并发保护。
7. 空闲链表能复用内存，但可能产生碎片。
