# 写者优先读写锁

本练习位于 `exercises/03_os_concurrency/05_rwlock/src/lib.rs`，使用一个
`AtomicU32` 和 `UnsafeCell<T>` 从零实现读写锁。它允许多个读者同时访问数据，
但写者访问时必须独占数据；当有写者等待时，新的读者不会被允许进入。

## 1. 读写锁的访问规则

读写锁需要满足以下互斥关系：

```text
多个读者可以同时持有读锁
写者持有写锁时，不能有任何读者
读者持有读锁时，写者必须等待
有写者等待时，新的读者不能进入
```

最后一条规则就是写者优先策略。它避免了读者持续到达时，写者一直无法
获得锁的情况。

## 2. 数据结构

```rust
pub struct RwLock<T> {
    state: AtomicU32,
    data: UnsafeCell<T>,
}
```

- `state`：同时保存读者数量和写者状态。
- `data`：保存被保护的数据。

`UnsafeCell` 只提供内部可变性，不提供线程安全。线程安全来自 `state` 的
原子更新和读写锁协议。

## 3. 状态位布局

```rust
const READER_MASK: u32 = (1 << 30) - 1;
const WRITER_HOLDING: u32 = 1 << 30;
const WRITER_WAITING: u32 = 1 << 31;
```

`state` 的布局如下：

```text
31                         30 29                         0
+----------------------------+----------------------------+
|       WRITER_WAITING       |       WRITER_HOLDING       | reader count
+----------------------------+----------------------------+
```

实际含义是：

| 位或字段 | 含义 |
|---|---|
| `state & READER_MASK` | 当前持有读锁的线程数量 |
| `WRITER_HOLDING` | 某个写者已经持有写锁 |
| `WRITER_WAITING` | 至少有一个写者正在等待 |

例如：

```text
state = 0
    没有读者，也没有写者

state = 3
    有三个读者

state = WRITER_WAITING | 5
    五个读者仍在锁内，同时有写者等待

state = WRITER_HOLDING
    写者独占持有锁
```

## 4. 获取读锁

读锁的核心逻辑是：

```rust
loop {
    let state = self.state.load(Ordering::Acquire);

    if state & (WRITER_HOLDING | WRITER_WAITING) != 0 {
        core::hint::spin_loop();
        continue;
    }

    if state & READER_MASK == READER_MASK {
        core::hint::spin_loop();
        continue;
    }

    if self.state.compare_exchange(
        state,
        state + 1,
        Ordering::AcqRel,
        Ordering::Acquire,
    ).is_ok() {
        break;
    }
}
```

每次尝试包含四步：

1. 使用 `Acquire` 读取当前状态。
2. 如果写者正在持有或等待，继续自旋。
3. 如果读者数量达到字段上限，继续自旋。
4. 使用 CAS 把读者数量加一。

CAS 的比较对象是完整的 `state`。如果另一个线程在读取和 CAS 之间修改了
状态，CAS 会失败，当前线程必须重新读取状态，不能基于旧值继续操作。

读锁成功后返回：

```rust
RwLockReadGuard { lock: self }
```

同一时刻，多个线程可以分别成功执行 `state -> state + 1`，因此读者之间
不会互相排斥。

## 5. 获取写锁

写者首先登记等待状态：

```rust
self.state.fetch_or(WRITER_WAITING, Ordering::Release);
```

这一步会阻止之后到达的读者进入。然后写者循环等待：

```rust
loop {
    let state = self.state.load(Ordering::Acquire);

    if state & (READER_MASK | WRITER_HOLDING) != 0 {
        core::hint::spin_loop();
        continue;
    }

    if self.state.compare_exchange(
        state,
        state | WRITER_HOLDING,
        Ordering::AcqRel,
        Ordering::Acquire,
    ).is_ok() {
        break;
    }
}
```

只有以下条件同时满足时，写者才能进入：

```text
读者数量为 0
WRITER_HOLDING 未设置
```

成功时，写者设置 `WRITER_HOLDING`，随后返回
`RwLockWriteGuard`。写者使用 `DerefMut` 获得唯一的 `&mut T`。

## 6. Guard 与 RAII

读锁和写锁都通过 Guard 管理生命周期：

```rust
pub struct RwLockReadGuard<'a, T> {
    lock: &'a RwLock<T>,
}

pub struct RwLockWriteGuard<'a, T> {
    lock: &'a RwLock<T>,
}
```

Guard 实现 `Deref`，因此可以像访问普通引用一样读取数据：

```rust
let guard = lock.read();
println!("{}", *guard);
```

写 Guard 额外实现 `DerefMut`：

```rust
let mut guard = lock.write();
*guard += 1;
```

离开作用域时，Rust 自动调用 `Drop`，不需要手动解锁。

## 7. 释放读锁

读 Guard 的 `Drop` 实现为：

```rust
self.lock.state.fetch_sub(1, Ordering::Release);
```

它只减少低位的读者计数，不会修改写者标志。最后一个读者退出后，
等待中的写者就有机会通过 CAS 设置 `WRITER_HOLDING`。

`fetch_sub` 是原子的，因此多个读者同时退出时不会发生读者计数更新丢失。

## 8. 释放写锁

写 Guard 的 `Drop` 实现为：

```rust
self.lock.state.fetch_and(
    !(WRITER_HOLDING | WRITER_WAITING),
    Ordering::Release,
);
```

它同时清除：

- `WRITER_HOLDING`：表示写者不再独占锁。
- `WRITER_WAITING`：表示当前写者已经完成等待和持有过程。

使用 `Release` 释放写锁，可以将写者在临界区内对数据的修改发布给之后
使用 `Acquire` 成功获取锁的读者或写者。

## 9. 内存序分析

### `Acquire`

用于读取状态和 CAS 失败路径。线程通过 Acquire 观察到其他线程释放的状态后，
可以读取其释放前完成的数据访问。

### `Release`

用于写者登记等待、读者退出和写者退出。它保证临界区内的数据操作不会被
重排到释放锁之后。

### `AcqRel`

用于成功的 CAS。CAS 同时读取旧状态并写入新状态，因此成功时既需要
Acquire 的获取语义，也需要 Release 的发布语义。

### CAS 失败序

失败的 CAS 只使用 `Acquire`。失败时没有获得锁，但需要重新观察其他线程
发布的状态，因此仍然需要获取语义。

## 10. `Send` 和 `Sync`

```rust
unsafe impl<T: Send> Send for RwLock<T> {}
unsafe impl<T: Send + Sync> Sync for RwLock<T> {}
```

- `Send` 要求 `T: Send`，表示锁及其数据可以转移到其他线程。
- `Sync` 要求 `T: Send + Sync`，表示多个线程可以共享同一个 `RwLock<T>`。

读锁返回共享引用，因此 `T` 必须支持跨线程共享。写锁返回可变引用，
而写者的独占状态保证同一时刻不会有其他引用访问数据。

这些 `unsafe impl` 的正确性依赖于锁协议本身。若绕过 Guard、错误修改
状态位或在未持锁时解引用 `UnsafeCell`，就可能产生数据竞争和未定义行为。

## 11. 自旋等待的适用范围

该实现使用 `core::hint::spin_loop()`，线程在等待期间不会休眠：

```rust
core::hint::spin_loop();
```

它适合临界区很短、锁很快释放的场景。若写者或读者可能长时间持锁，
自旋线程会持续消耗 CPU，生产环境通常需要结合阻塞唤醒机制。

另外，当前实现用一个 `WRITER_WAITING` 位表示“至少有写者等待”，
并没有单独记录等待写者数量。它足以表达写者优先的基本教学协议，但若要
实现严格的多写者公平排队，通常还需要等待计数器、队列或更完整的调度策略。

## 12. 核心要点

1. 低 30 位保存读者数量，高两位保存写者状态。
2. 读者只有在没有写者持有、没有写者等待时才能进入。
3. 写者先设置 `WRITER_WAITING`，阻止新读者进入。
4. CAS 保证读者计数增加和写者状态转换不会丢失更新。
5. 读 Guard 退出时递减读者计数。
6. 写 Guard 退出时清除写者状态。
7. `Acquire`、`Release` 和 `AcqRel` 共同保证状态与数据的可见性。
8. Guard 的 `Drop` 实现提供自动解锁和 panic 安全性。
9. 自旋锁适合短等待，长等待应考虑阻塞式同步机制。
