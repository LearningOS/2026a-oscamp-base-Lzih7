# Tokio 异步任务

本练习位于 `exercises/05_async_programming/02_tokio_tasks/src/lib.rs`，主要学习
如何使用 `tokio::spawn` 创建异步任务，并通过 `JoinHandle` 等待任务完成。

这个练习的核心不是“把代码写成 async”，而是理解多个异步任务如何被运行时
并发调度。

## 1. `tokio::spawn`

`tokio::spawn` 用来创建一个 Tokio 异步任务：

```rust
let handle = tokio::spawn(async move {
    i * i
});
```

它的作用可以理解为：

```text
把这个 async 代码块交给 Tokio 运行时调度执行
```

调用 `tokio::spawn` 后，任务不会阻塞当前函数。当前函数会立刻拿到一个
`JoinHandle`，之后可以通过 `.await` 等待这个任务的结果。

## 2. `JoinHandle`

文件中引入了：

```rust
use tokio::task::JoinHandle;
```

`JoinHandle<T>` 表示一个已经被提交给 Tokio 运行时的任务句柄。任务完成后，
可以通过：

```rust
handle.await
```

取得结果。

不过 `handle.await` 返回的不是直接的任务返回值，而是：

```rust
Result<T, JoinError>
```

所以代码中使用：

```rust
handle.await.unwrap()
```

含义是：

```text
等待任务完成
如果任务正常完成，取出它的返回值
如果任务 panic 或被取消，则 unwrap 触发 panic
```

在本练习的测试场景中，任务逻辑很简单，不会主动 panic，因此可以直接
使用 `unwrap()`。

## 3. `async move`

创建任务时使用：

```rust
tokio::spawn(async move {
    i * i
})
```

这里的 `move` 很重要。它表示把外部变量 `i` 的所有权移动进异步任务中。

如果没有 `move`，异步任务可能只是借用外部变量。但任务的执行时间由运行时
决定，可能在当前循环的一轮结束后才真正运行。为了避免悬垂引用，Tokio
要求被 `spawn` 的任务通常拥有自己需要的数据。

因此：

```rust
let handle = tokio::spawn(async move {
    i * i
});
```

表示每个任务都拥有自己那一轮循环里的 `i`。

## 4. 并发计算平方

函数：

```rust
pub async fn concurrent_squares(n: usize) -> Vec<usize>
```

目标是并发计算 `0..n` 中每个数字的平方，并按顺序返回结果。

实现流程：

```rust
let mut handles = Vec::new();
for i in 0..n {
    let handle = tokio::spawn(async move {
        i * i
    });
    handles.push(handle);
}
```

这段代码先创建所有任务。每个任务负责计算一个平方值：

```text
任务 0：0 * 0
任务 1：1 * 1
任务 2：2 * 2
...
```

随后再等待所有任务完成：

```rust
let mut results = Vec::new();
for handle in handles {
    results.push(handle.await.unwrap());
}
```

因为 `handles` 的顺序与创建任务的顺序一致，所以结果也会按 `0..n` 的顺序
被收集。

例如：

```rust
concurrent_squares(5).await
```

返回：

```rust
vec![0, 1, 4, 9, 16]
```

## 5. 为什么先收集 Handle，再 await

关键点是：应该先把任务全部创建出来，再等待它们。

当前写法：

```rust
for i in 0..n {
    let handle = tokio::spawn(async move {
        i * i
    });
    handles.push(handle);
}

for handle in handles {
    results.push(handle.await.unwrap());
}
```

这表示：

```text
先启动所有任务
再逐个等待结果
```

如果写成：

```rust
for i in 0..n {
    let result = tokio::spawn(async move {
        i * i
    }).await.unwrap();
    results.push(result);
}
```

那就变成：

```text
启动任务 0，等任务 0 完成
启动任务 1，等任务 1 完成
启动任务 2，等任务 2 完成
```

这样并发性会明显降低，尤其是任务内部有等待操作时。

## 6. 并发 sleep 任务

函数：

```rust
pub async fn parallel_sleep_tasks(n: usize, duration_ms: u64) -> Vec<usize>
```

每个任务都会：

```rust
sleep(Duration::from_millis(duration_ms)).await;
i
```

也就是等待指定时间后返回自己的任务编号。

创建任务：

```rust
let mut handles = Vec::new();
for i in 0..n {
    let handle = tokio::spawn(async move {
        sleep(Duration::from_millis(duration_ms)).await;
        i
    });
    handles.push(handle);
}
```

如果有 5 个任务，每个任务 sleep 100ms，并发执行时总耗时应该接近：

```text
100ms
```

而不是：

```text
5 * 100ms = 500ms
```

测试中检查：

```rust
assert!(elapsed.as_millis() < 400);
```

它不是要求精确等于 100ms，而是留出调度和测试环境开销。只要明显小于
串行执行的 500ms，就说明任务确实在并发运行。

## 7. 为什么结果要排序

在 `parallel_sleep_tasks` 中：

```rust
results.sort();
```

任务是并发执行的。虽然当前代码按 `handles` 的顺序 await，通常结果顺序
会保持为 `0..n`，但排序可以让函数更明确地表达：

```text
只关心所有任务都完成并返回了自己的编号，不依赖任务完成顺序
```

如果以后改成 `JoinSet` 或其他“谁先完成先收集”的方式，排序也能让测试结果
保持稳定。

## 8. `sleep(...).await` 不会阻塞线程

这里使用的是：

```rust
tokio::time::sleep(Duration::from_millis(duration_ms)).await;
```

它不是普通的线程阻塞睡眠。普通同步睡眠类似：

```rust
std::thread::sleep(...)
```

会阻塞当前操作系统线程。

Tokio 的 `sleep(...).await` 会让当前异步任务进入等待状态，把执行权还给
运行时。运行时可以继续执行其他任务。等定时器到期后，任务会被唤醒并继续
执行。

所以多个异步 sleep 可以重叠进行。

## 9. `JoinHandle` 的类型

对于平方任务：

```rust
let handle = tokio::spawn(async move {
    i * i
});
```

`handle` 的类型可以理解为：

```rust
JoinHandle<usize>
```

因为 async 代码块最终返回的是 `usize`。

对于 sleep 任务：

```rust
let handle = tokio::spawn(async move {
    sleep(Duration::from_millis(duration_ms)).await;
    i
});
```

返回值同样是 `usize`，所以也是：

```rust
JoinHandle<usize>
```

如果任务没有返回值：

```rust
tokio::spawn(async move {
    println!("hello");
});
```

那么句柄类型就是：

```rust
JoinHandle<()>
```

## 10. `n = 0` 的情况

当调用：

```rust
concurrent_squares(0).await
```

循环：

```rust
for i in 0..n
```

不会执行任何一次，因此：

```rust
handles = []
results = []
```

最终返回空数组：

```rust
vec![]
```

这就是 `test_squares_zero` 检查的内容。

## 11. 核心要点

1. `tokio::spawn` 创建异步任务并交给运行时调度。
2. `JoinHandle<T>` 用来等待任务完成并取得返回值。
3. `handle.await` 返回 `Result<T, JoinError>`。
4. `async move` 将任务需要的数据移动进任务内部。
5. 先创建所有任务，再 await，才能体现并发执行。
6. `tokio::time::sleep(...).await` 不会阻塞操作系统线程。
7. 多个 sleep 任务可以重叠等待，总耗时接近单个任务耗时。
8. 收集结果时可以按 handle 顺序，也可以排序保证结果稳定。
