# 原子内存序与线程同步

本练习位于 `exercises/03_os_concurrency/02_atomic_ordering/src/lib.rs`，通过 `FlagChannel` 和 `OnceCell` 学习原子操作的内存序，以及线程之间如何安全地发布数据。

## 1. 原子性与可见性

原子类型保证单次操作不会被拆分或交错破坏，但“操作是原子的”不等于“其他线程一定能按预期看到相关数据”。

例如生产者先写入普通数据，再设置一个 ready 标志。消费者只有在正确读取 ready 后，才能确认数据已经完成写入。这个顺序关系由内存序建立。

## 2. `FlagChannel` 的发布流程

生产者执行：

```rust
self.data.store(value, Ordering::Relaxed);
self.ready.store(true, Ordering::Release);
```

这里先写数据，再用 `Release` 写入 ready。Release 表示在它之前的内存操作不能被重新排列到它之后，并向读取同一原子变量的 Acquire 操作发布这些写入。

消费者执行：

```rust
while !self.ready.load(Ordering::Acquire) {
    std::hint::spin_loop();
}
let value = self.data.load(Ordering::Relaxed);
```

消费者通过 Acquire 读取到 `true` 后，就能看到生产者在 Release 之前完成的数据写入。因此 data 本身可以使用 Relaxed，ready 负责建立同步关系。

## 3. 自旋等待

`consume` 在 ready 为 `false` 时循环检查：

```rust
while !self.ready.load(Ordering::Acquire) {
    std::hint::spin_loop();
}
```

这种方式称为 spin-wait（自旋等待）。线程不会休眠，而是反复读取原子标志。`spin_loop()` 向处理器提示当前线程正在等待，适合预计等待时间很短的场景。

如果等待时间可能较长，自旋会持续占用 CPU，此时通常应使用条件变量、channel 或其他阻塞同步工具。

## 4. 常见内存序

### `Relaxed`

只保证原子性，不建立线程间的先后关系。适合：

- 独立计数器。
- 不依赖其他数据的状态统计。
- 已由其他原子变量建立同步关系的数据读取。

### `Release`

常用于发布数据。当前线程在 Release 操作之前的读写，会对另一个线程的 Acquire 读取可见。

### `Acquire`

常用于接收已发布的数据。读取到对应的 Release 写入后，后续读写可以看到发布线程之前的结果。

### `AcqRel`

同时具有 Acquire 和 Release 语义，常用于成功执行的 CAS 读-改-写操作。

### `SeqCst`

提供顺序一致性，所有线程观察到的顺序保持一致。它最容易理解，但约束更强，可能比更弱的内存序带来更高成本。

## 5. `OnceCell` 的一次初始化

`OnceCell` 使用两个原子标志：

```rust
pub struct OnceCell {
    initialized: AtomicBool,
    ready: AtomicBool,
    value: AtomicU32,
}
```

- `initialized`：通过 CAS 竞争初始化权，保证只有一个线程成功。
- `value`：保存初始化数据。
- `ready`：表示 value 已经写入完成，可以被读取。

初始化过程：

```rust
if self.initialized
    .compare_exchange(false, true, Ordering::SeqCst, Ordering::SeqCst)
    .is_ok()
{
    self.value.store(val, Ordering::Relaxed);
    self.ready.store(true, Ordering::Release);
    true
} else {
    false
}
```

CAS 成功的线程拥有初始化权。它先写入 value，再用 Release 设置 ready。

读取过程：

```rust
if self.ready.load(Ordering::Acquire) {
    Some(self.value.load(Ordering::Relaxed))
} else {
    None
}
```

只有读取到 ready 为 `true`，调用者才会读取 value。Acquire 与 Release 配对，确保 value 的写入对读取线程可见。

## 6. 为什么不能只使用 `initialized`

如果初始化线程先把 `initialized` 设置为 `true`，之后才写入 value，另一个线程可能观察到：

```text
initialized == true
value 还没有写入完成
```

此时 `get()` 会过早返回无效或旧数据。因此需要将“已经抢到初始化权”和“值已经可以读取”区分开。

`initialized` 负责一次性竞争，`ready` 负责数据发布。两个状态承担不同职责。

## 7. CAS 的作用

CAS（Compare-And-Exchange）会比较当前值和期望值：

```text
当前值 == expected：
    写入 new，并返回成功
否则：
    不修改，并返回失败
```

多个线程同时调用 `init` 时，只有一个线程能把 `initialized` 从 `false` 改为 `true`，其他线程收到 `Err` 并返回 `false`。

## 8. `reset` 的内存序

练习中的 reset 使用 Relaxed：

```rust
self.ready.store(false, Ordering::Relaxed);
self.data.store(0, Ordering::Relaxed);
```

它只是清除当前状态，并没有用于向其他线程发布一组新的共享数据。若 reset 与 produce/consume 并发执行，则还需要额外的协议保证，不能只依赖原子变量本身。

## 9. 核心要点

1. 原子性保证单次操作安全，内存序负责建立可见性和先后关系。
2. Release 用于发布数据，Acquire 用于接收已发布的数据。
3. `FlagChannel` 先写 data，再 Release 写 ready。
4. `OnceCell` 使用 CAS 保证只有一个线程初始化。
5. `initialized` 表示获得初始化权，`ready` 表示数据已经完成发布。
6. Relaxed 不建立额外同步，只保证原子读写。
7. 自旋适合短等待，长等待应考虑阻塞式同步工具。
