# 文件描述符表

本练习位于 `exercises/02_no_std_dev/05_fd_table/src/lib.rs`，实现一个简化的文件描述符表。操作系统通常通过整数 fd 将用户程序与内核中的文件对象关联起来。

```text
fd table:
  0 -> Stdin
  1 -> Stdout
  2 -> Stderr
  3 -> File
  4 -> empty
```

## 1. 数据结构

本练习使用：

```rust
files: Vec<Option<Arc<dyn File>>>
```

vector 的下标就是 fd 编号：

- `Some(file)` 表示该 fd 当前已分配。
- `None` 表示该 fd 已关闭或从未使用。

这种结构称为稀疏表。关闭 fd 后不会移动其他元素，因此已有 fd 编号保持稳定。

## 2. `File` trait 与 trait object

所有文件类型都实现统一接口：

```rust
pub trait File: Send + Sync {
    fn read(&self, buf: &mut [u8]) -> isize;
    fn write(&self, buf: &[u8]) -> isize;
}
```

`Arc<dyn File>` 表示指向某个实现了 `File` trait 的对象的共享指针。调用者不需要知道具体类型是普通文件、管道还是 socket，只需要通过 `read` 和 `write` 使用它。

`dyn File` 是动态分发的 trait object；`Arc` 负责共享所有权和引用计数。`File: Send + Sync` 表示文件对象可以安全地在线程间传递和共享。

## 3. 分配 fd

分配时从下标 `0` 开始查找第一个空槽：

```rust
let mut i = 0;
while i < self.files.len() && self.files[i].is_some() {
    i += 1;
}
```

找到空槽后复用它；如果已经遍历到 vector 末尾，则追加新元素：

```rust
if i == self.files.len() {
    self.files.push(Some(file));
} else {
    self.files[i] = Some(file);
}
```

这种策略保证返回最小的可用 fd。例如关闭 fd `1` 后，下一次分配会优先返回 `1`，而不是继续使用更大的编号。

查找空闲槽的时间复杂度是 `O(n)`。实际内核可能使用位图、空闲 fd 链表或其他索引结构优化查找。

## 4. 获取文件对象

```rust
pub fn get(&self, fd: usize) -> Option<Arc<dyn File>> {
    self.files.get(fd).cloned().unwrap_or(None)
}
```

`get()` 的行为如下：

- fd 在范围内且已分配：返回 `Some(Arc<dyn File>)`。
- fd 超出 vector 范围：返回 `None`。
- fd 对应槽位为空：返回 `None`。

调用 `.cloned()` 会增加 `Arc` 的引用计数，返回一个新的共享句柄。它不会复制底层文件对象。

## 5. 关闭 fd

关闭操作把对应槽位设置为 `None`：

```rust
match self.files.get_mut(fd) {
    Some(slot @ Some(_)) => {
        *slot = None;
        true
    }
    _ => false,
}
```

只有实际占用的 fd 才能关闭成功。关闭不存在或已经关闭的 fd 返回 `false`。

把 `Some(Arc<dyn File>)` 替换为 `None` 后，该表持有的 `Arc` 引用会被释放。如果这是最后一个引用，底层文件对象也会被销毁。

## 6. 统计已分配 fd

```rust
self.files.iter().filter(|slot| slot.is_some()).count()
```

`count()` 返回当前仍然占用的 fd 数量，而不是 vector 的长度。vector 可能包含已经关闭的空槽，因此：

```text
files.len()     = 4
Some 的数量     = 2
count()         = 2
```

## 7. 所有权与资源管理

fd 表本身拥有存储在槽位中的 `Arc` 引用，但调用 `get()` 后得到的对象也可以被其他代码持有：

```rust
let file = table.get(fd).unwrap();
```

此时即使调用 `table.close(fd)`，`file` 变量仍然持有底层对象。关闭 fd 只会移除“通过这个 fd 访问对象”的映射，不一定立即销毁文件对象。

这体现了 fd 表和文件对象生命周期的区别：

- fd 是索引和访问入口。
- `Arc<dyn File>` 管理实际对象的共享生命周期。

## 8. 并发使用注意事项

当前 `FdTable` 的 `alloc`、`close` 等方法需要 `&mut self`，因此同一时刻只能由一个可变借用者修改表。它本身不是并发安全容器。

如果多个线程需要共享同一张 fd 表，可以在外部使用：

```rust
Arc<Mutex<FdTable>>
```

读写操作都应遵循统一的锁策略，避免 fd 分配、关闭和查找之间产生竞态。

## 9. 核心要点

1. `Vec<Option<Arc<dyn File>>>` 用下标直接映射 fd 编号。
2. `Some` 表示已分配，`None` 表示空闲或已关闭。
3. 分配时优先复用编号最小的空槽。
4. 表满时追加新槽位，避免数组越界。
5. `get()` 克隆的是 `Arc`，不会复制底层文件对象。
6. `close()` 清除 fd 映射，并减少一个 `Arc` 引用。
7. `count()` 统计实际占用的槽位数量，而不是 vector 长度。
