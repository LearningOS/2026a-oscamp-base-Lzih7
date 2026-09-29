# 自旋锁

本练习位于 `exercises/03_os_concurrency/03_spinlock/src/lib.rs`，实现一个基础的自旋锁。自旋锁在锁被占用时不会让线程阻塞睡眠，而是持续尝试获取锁。

## 1. 数据结构

```rust
pub struct SpinLock<T> {
    locked: AtomicBool,
    data: UnsafeCell<T>,
}
```

- `locked`：表示锁是否已经被占用。
- `data`：保存受保护的数据。

`AtomicBool` 负责线程之间安全地竞争锁状态，`UnsafeCell<T>` 允许通过共享引用访问内部可变数据。

## 2. 使用 CAS 获取锁

获取锁时，需要把状态从 `false` 原子地改为 `true`：

```rust
self.locked.compare_exchange(
    false,
    true,
    Ordering::Acquire,
    Ordering::Relaxed,
)
```

CAS 的含义是：

```text
当前值为 false：
    修改为 true，当前线程获得锁
当前值为 true：
    修改失败，锁仍被其他线程占用
```

只有一个线程能够成功地把 `false` 改成 `true`。其他线程必须继续等待，不能因为 CAS 失败就直接访问数据。

## 3. 自旋等待

```rust
loop {
    match self.locked.compare_exchange(false, true, ...) {
        Ok(false) => break,
        Err(_) => core::hint::spin_loop(),
        Ok(true) => unreachable!(),
    }
}
```

失败后调用 `spin_loop()`，向处理器提示当前线程正在进行短时间自旋等待。它可能帮助处理器降低功耗或优化超线程执行。

自旋锁适合临界区很短、持锁线程很快释放锁的场景。如果锁可能长时间被占用，自旋会持续消耗 CPU，此时互斥锁或阻塞式同步工具通常更合适。

## 4. 内存序

### 获取锁使用 `Acquire`

成功获取锁时使用 `Acquire`，保证当前线程之后的读取和写入不会被移动到获取锁之前，并能看到前一个持锁线程释放前发布的数据。

### 释放锁使用 `Release`

```rust
self.locked.store(false, Ordering::Release);
```

释放锁时使用 `Release`，保证临界区中的数据访问先于解锁操作完成。下一个使用 Acquire 成功获得锁的线程可以看到这些更新。

Acquire 和 Release 配对形成同步关系：

```text
线程 A：修改 data -> Release 解锁
线程 B：Acquire 加锁 -> 读取 data
```

## 5. `UnsafeCell` 与内部可变性

Rust 默认不允许通过 `&self` 修改数据。`UnsafeCell<T>` 是标准库提供的底层内部可变性原语：

```rust
unsafe { &mut *self.data.get() }
```

只有在已经成功持有锁时，才能将内部指针转换为 `&mut T`。锁保证同一时刻只有一个线程获得这个可变引用。

`UnsafeCell` 本身不提供线程安全，它只是允许绕过普通共享引用的不可变限制。真正的同步责任由 `AtomicBool` 和调用协议承担。

## 6. `try_lock`

`try_lock` 只进行一次 CAS：

```rust
if self.locked.compare_exchange(false, true, ...) == Ok(false) {
    Some(unsafe { &mut *self.data.get() })
} else {
    None
}
```

它不会等待：

- 成功获取锁：返回 `Some(&mut T)`。
- 锁已占用：立即返回 `None`。

这种接口适合调用者可以选择其他工作或稍后重试的场景。

## 7. 手动解锁的安全要求

当前练习的接口要求调用者手动调用：

```rust
let data = lock.lock();
*data += 1;
lock.unlock();
```

必须遵守：

1. 只有成功获取锁后才能调用 `unlock()`。
2. 使用完返回的 `&mut T` 后才能解锁。
3. 不能忘记解锁，否则其他线程会无限自旋。
4. 不能重复解锁，否则锁状态会被破坏。
5. 不能在解锁后继续使用之前取得的可变引用。

生产级实现通常使用 RAII guard，在 guard 的 `Drop` 中自动解锁，以减少忘记解锁的风险。

## 8. `Send` 和 `Sync`

```rust
unsafe impl<T: Send> Sync for SpinLock<T> {}
unsafe impl<T: Send> Send for SpinLock<T> {}
```

如果 `T: Send`，说明 `T` 可以在线程之间转移。锁通过互斥访问保证同一时刻只有一个线程操作内部数据，因此 `SpinLock<T>` 可以在线程间共享。

这些 `unsafe impl` 要求实现者确保锁的同步逻辑确实正确；错误的锁实现可能导致数据竞争和未定义行为。

## 9. 核心要点

1. `AtomicBool` 保存锁状态，CAS 用于竞争获取锁。
2. CAS 失败时必须继续等待，不能直接访问受保护数据。
3. `Acquire` 获取锁，`Release` 释放锁，二者建立数据可见性。
4. `spin_loop()` 适合短时间等待，但长时间自旋会浪费 CPU。
5. `UnsafeCell` 提供内部可变性，但不负责同步。
6. 手动 `unlock()` 必须严格遵守调用顺序。
7. RAII guard 是更安全的生产级解锁方式。
