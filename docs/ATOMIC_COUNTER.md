# 原子计数器

本练习位于 `exercises/03_os_concurrency/01_atomic_counter/src/lib.rs`，使用 `AtomicU64` 实现一个无需互斥锁的线程安全计数器。

## 1. 为什么使用原子类型

普通的 `u64` 自增通常包含三个步骤：

```text
读取旧值 -> 加 1 -> 写回新值
```

如果多个线程同时执行，可能出现两个线程读取相同旧值，导致其中一次更新被覆盖。

`AtomicU64` 提供不可分割的读写和更新操作。多个线程可以同时访问同一个原子变量，而不会产生数据竞争。

```rust
pub struct AtomicCounter {
    value: AtomicU64,
}
```

## 2. `fetch_add` 和 `fetch_sub`

递增和递减使用：

```rust
self.value.fetch_add(1, Ordering::Relaxed)
self.value.fetch_sub(1, Ordering::Relaxed)
```

这两个函数都会原子地修改值，但返回的是修改前的旧值：

```text
初始值 = 5
fetch_add(1) 返回 5
修改后值 = 6
```

因此 `increment()` 的第一次调用返回 `0`，第二次调用返回 `1`。

`fetch_sub` 的行为类似：

```text
初始值 = 5
fetch_sub(1) 返回 5
修改后值 = 4
```

## 3. `load`

读取当前值使用：

```rust
self.value.load(Ordering::Relaxed)
```

`load` 是原子读取，不会读到被并发写入破坏的中间状态。对于单纯的计数和读取，如果不需要通过计数器同步其他数据，可以使用 `Relaxed`。

## 4. 内存序 `Ordering`

原子操作不仅要保证操作本身不可分割，还要规定不同线程观察这些操作的顺序。

### `Relaxed`

`Relaxed` 只保证原子性，不建立额外的线程间同步关系。适合：

- 统计访问次数。
- 统计任务数量。
- 只关心最终数值的计数器。

例如，多个线程调用 `increment()` 时，每次加法都不会丢失，但计数器本身不负责发布其他普通变量的数据。

### `Acquire` 与 `Release`

`Acquire` 和 `Release` 常用于一个线程发布数据，另一个线程读取数据的场景。它们比 `SeqCst` 更灵活，但需要结合具体同步关系分析。

### `AcqRel`

`AcqRel` 同时具备获取和释放语义，常用于成功执行读-改-写操作的 CAS。

## 5. CAS 操作

`compare_exchange` 的基本形式是：

```rust
self.value.compare_exchange(
    expected,
    new_val,
    Ordering::AcqRel,
    Ordering::Acquire,
)
```

它会执行以下逻辑：

```text
如果当前值 == expected：
    原子地写入 new_val，返回 Ok(expected)
否则：
    不修改，返回 Err(实际当前值)
```

成功示例：

```text
当前值 = 10
expected = 10
new_val = 20
结果 = Ok(10)
当前值 = 20
```

失败示例：

```text
当前值 = 10
expected = 5
new_val = 20
结果 = Err(10)
当前值仍为 10
```

CAS 能够让线程在不加锁的情况下，根据“值没有被其他线程修改”这一条件提交更新。

## 6. CAS 循环

复杂的原子更新通常需要循环：

```rust
loop {
    let current = counter.get();
    let new_value = current * multiplier;

    match counter.compare_and_swap(current, new_value) {
        Ok(_) => break current,
        Err(actual) => {
            // 其他线程已经修改了值，使用最新值重试
            let _ = actual;
        }
    }
}
```

如果 CAS 失败，说明读取旧值和提交更新之间发生了竞争。此时不能继续使用旧的计算结果，而应读取最新值，重新计算目标值，再次尝试。

例如两个线程同时将 `10` 乘以 `2`：

```text
线程 A 读取 10
线程 B 读取 10
线程 A 成功写入 20
线程 B CAS 失败，发现实际值为 20
线程 B 重新计算 20 * 2，最终写入 40
```

## 7. 无锁不等于无竞争

原子计数器不使用 `Mutex`，但线程之间仍然可能竞争同一个原子变量。CAS 失败就是竞争的表现。

无锁算法的特点是：

- 不会因为某个线程持有锁而阻塞其他线程。
- 失败线程需要重试。
- 高竞争场景下可能产生较多重试。
- 必须正确选择内存序。

## 8. `Arc` 与线程共享

测试中使用 `Arc` 将计数器共享给多个线程：

```rust
let counter = Arc::new(AtomicCounter::new(0));
let c = Arc::clone(&counter);
```

`Arc` 只负责共享 `AtomicCounter` 的所有权；真正保证计数更新安全的是内部的 `AtomicU64`。

## 9. 核心要点

1. `AtomicU64` 提供线程安全的原子读写。
2. `fetch_add` 和 `fetch_sub` 返回修改前的值。
3. `Relaxed` 保证原子性，但不建立额外的数据同步关系。
4. CAS 只有在当前值等于预期值时才提交更新。
5. CAS 失败后必须用最新值重新计算并重试。
6. `Arc` 共享对象所有权，原子类型负责内部并发更新。
7. 无锁算法仍然可能发生竞争，只是通过重试处理竞争。
