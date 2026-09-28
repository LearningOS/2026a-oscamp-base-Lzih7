# `Arc<Mutex<T>>` 共享状态

本练习位于 `exercises/01_concurrency_sync/02_mutex_counter/src/lib.rs`，通过两个函数学习多个线程如何安全地访问和修改同一份数据：

- `concurrent_counter`：多个线程共同递增一个计数器。
- `concurrent_collect`：多个线程向共享 vector 中添加自己的编号。

## 1. 为什么需要同步

多个线程同时修改同一个变量时，如果没有同步机制，可能出现数据竞争。例如两个线程同时读取计数器 `10`，分别计算出 `11`，最后只写入一次 `11`，导致一次递增丢失。

`Mutex<T>` 提供互斥访问：同一时刻只有持有锁的线程可以访问内部数据。

## 2. `Arc<Mutex<T>>`

```rust
let counter = Arc::new(Mutex::new(0usize));
```

这里有两层作用：

- `Mutex<T>` 保护 `T`，保证修改操作不会同时发生。
- `Arc<T>` 提供线程安全的引用计数，允许多个线程共同持有同一个 mutex。

把 `Arc` 克隆后，得到的是指向同一份数据的新引用，而不是复制数据本身：

```rust
let counter_for_thread = Arc::clone(&counter);
```

每个线程都应持有自己的 `Arc` 引用，并通过 `move` 将该引用移动进线程闭包。

## 3. 获取和使用锁

```rust
let mut counter = counter.lock().unwrap();
*counter += 1;
```

`lock()` 成功后返回 `MutexGuard`。它可以像内部数据一样被读写；当 guard 离开作用域时，锁会自动释放。

`lock()` 返回 `Result`，因为如果之前持锁的线程 panic，mutex 可能进入 poisoned 状态。练习使用 `unwrap()` 表示遇到这种情况直接传播 panic；实际程序也可以根据业务需要恢复数据。

在计数器练习中，一个线程获取锁后执行多次递增：

```rust
for _ in 0..count_per_thread {
    *counter += 1;
}
```

这种写法是正确的，但会让该线程在整个循环期间持有锁。若临界区较大，其他线程需要等待，因此应尽量缩短持锁时间。

## 4. 等待所有线程结束

创建线程时保存每个 `JoinHandle`：

```rust
for thread in threads {
    thread.join().unwrap();
}
```

只有等待所有线程结束后，主线程才能读取最终结果。`join()` 同时也会报告子线程是否发生 panic。

## 5. 共享 vector 的并发写入

`concurrent_collect` 为每个线程分配一个编号：

```rust
for i in 0..n_threads {
    let counter = Arc::clone(&counter);
    threads.push(thread::spawn(move || {
        let mut values = counter.lock().unwrap();
        values.push(i);
    }));
}
```

线程执行顺序是不确定的，所以 vector 的初始顺序不能依赖线程编号。所有线程结束后调用：

```rust
result.sort();
```

即可得到稳定的升序结果。

## 6. `MutexGuard` 与返回值

不能直接从借用的 `MutexGuard` 中移动出 vector：

```rust
// 错误：试图从 guard 中移出 Vec
let result = *counter.lock().unwrap();
```

本练习使用克隆的方式取得结果：

```rust
let mut result = counter.lock().unwrap().clone();
```

锁内的 vector 实现了 `Clone`，因此可以复制出一份独立的 vector，随后释放锁并排序。对于大型数据，也可以在确认没有其他 `Arc` 引用后使用 `Arc::try_unwrap`，避免不必要的复制。

## 7. 核心要点

1. `Arc` 负责在线程间共享所有权，`Mutex` 负责保护共享数据。
2. 修改 mutex 内的数据必须先通过 `lock()` 获取 `MutexGuard`。
3. guard 离开作用域时自动释放锁。
4. 必须通过 `join()` 等待工作线程结束，再读取最终状态。
5. 并发结果的顺序通常不确定，需要显式排序或使用其他顺序控制。
6. 设计临界区时应尽量少持有锁，减少线程等待。
