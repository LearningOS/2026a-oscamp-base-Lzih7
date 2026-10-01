# Tokio Select 与超时控制

本练习位于 `exercises/05_async_programming/04_select_timeout/src/lib.rs`，通过
`tokio::select!` 和 `tokio::time::sleep` 实现异步操作竞争与超时控制。

核心思路是同时轮询多个异步分支，并在其中一个分支完成后返回对应结果。

## 1. `tokio::select!`

基本语法如下：

```rust
tokio::select! {
    value = future_a => {
        // future_a 先完成时执行
    }
    value = future_b => {
        // future_b 先完成时执行
    }
}
```

`select!` 会在当前异步任务中轮询多个 Future。某个分支先返回
`Poll::Ready` 后，执行该分支的处理代码，整个 `select!` 表达式结束。

这里的“同时等待”不表示每个 Future 都运行在独立线程上。它表示 Tokio
运行时在同一个异步任务中推进多个 Future；当 Future 等待定时器或 IO 时，
任务可以让出执行权。

## 2. `with_timeout`

函数签名：

```rust
pub async fn with_timeout<F, T>(future: F, timeout_ms: u64) -> Option<T>
where
    F: Future<Output = T>,
```

该函数接收一个输出类型为 `T` 的 Future，并返回：

```text
Some(value)：原 Future 在超时前完成
None：计时器先完成，操作超时
```

实现：

```rust
tokio::select! {
    result = future => Some(result),
    _ = sleep(Duration::from_millis(timeout_ms)) => None,
}
```

两个分支分别等待：

1. `future`：调用方提供的异步操作。
2. `sleep(...)`：等待指定毫秒数的计时器。

如果 `future` 先完成，返回 `Some(result)`；如果 sleep 先完成，返回
`None`。

## 3. 超时分支的时间单位

参数 `timeout_ms` 的单位是毫秒：

```rust
Duration::from_millis(timeout_ms)
```

例如：

```rust
with_timeout(operation, 50).await
```

表示最多等待 50 毫秒。

`tokio::time::sleep` 是异步计时器。计时期间，它不会阻塞当前操作系统线程；
当前任务可以挂起，让 Tokio 运行时执行其他任务。计时结束后，定时器会唤醒
对应任务，使 `select!` 再次检查各分支。

## 4. `race`

函数签名：

```rust
pub async fn race<F1, F2, T>(f1: F1, f2: F2) -> T
where
    F1: Future<Output = T>,
    F2: Future<Output = T>,
```

两个 Future 必须返回相同的类型 `T`，这样无论哪个先完成，函数都能返回
统一类型。

实现：

```rust
tokio::select! {
    result = f1 => result,
    result = f2 => result,
}
```

先完成的分支直接提供最终结果。例如：

```rust
let result = race(
    async {
        sleep(Duration::from_millis(10)).await;
        "fast"
    },
    async {
        sleep(Duration::from_millis(200)).await;
        "slow"
    },
)
.await;
```

第一个 Future 通常先完成，因此结果为 `"fast"`。

## 5. 未获胜分支的取消

默认情况下，`select!` 结束时，其他尚未完成的分支 Future 会被丢弃。
这称为取消（cancellation）。

例如超时分支先完成时：

```text
sleep 完成
with_timeout 返回 None
传入的 future 不再继续被轮询，并在离开 select! 时被丢弃
```

同理，`race` 中未获胜的 Future 也会被丢弃。

取消 Future 不会自动撤销它在此前已经完成的外部副作用。例如，一个 Future
可能在等待前已经写入文件、发送网络请求或修改共享状态；丢弃 Future 不会
自动回滚这些操作。设计可取消操作时，应考虑取消发生时的资源清理和状态一致性。

## 6. `select!` 与 `tokio::time::timeout`

本练习也可以使用 Tokio 的 `timeout` 函数表达超时：

```rust
match tokio::time::timeout(
    Duration::from_millis(timeout_ms),
    future,
)
.await
{
    Ok(value) => Some(value),
    Err(_) => None,
}
```

两种写法的侧重点不同：

| 写法 | 适用情况 |
|---|---|
| `tokio::select!` | 需要在多个异步分支之间选择，或自定义多个分支的处理 |
| `tokio::time::timeout` | 只需要给一个 Future 添加超时限制 |

`timeout` 的结果类型是 `Result<T, Elapsed>`，需要转换成练习要求的
`Option<T>`。

## 7. `select!` 的分支顺序与同时完成

如果多个分支在同一次轮询中都已经可以完成，`select!` 会根据宏的分支选择
规则决定执行其中一个分支。因此不应把代码正确性建立在“两个 Future 同时
完成时必定选择某个固定分支”的假设上。

本练习的测试使用明显不同的 sleep 时长，让一个分支清楚地先完成：

```text
快速分支：10ms
慢速分支：200ms
```

这样测试验证的是竞争结果，而不是边界时刻的优先级。

## 8. 测试流程

### 操作在超时前完成

```rust
let result = with_timeout(async { 42 }, 100).await;
assert_eq!(result, Some(42));
```

Future 立即返回 `42`，早于 100ms 计时器，结果为 `Some(42)`。

### 操作超过时限

```rust
let result = with_timeout(
    async {
        sleep(Duration::from_millis(200)).await;
        42
    },
    50,
)
.await;
```

操作需要约 200ms，计时器在 50ms 后先完成，因此结果为 `None`，慢操作
Future 被取消。

### 两个 Future 竞争

测试分别安排 10ms 和 200ms 的延迟，验证先完成的 Future 决定 `race`
返回值。交换两个分支的位置后，结果仍应来自较快的 Future。

## 9. 常见注意事项

### 输入 Future 需要满足 `Future`

泛型约束：

```rust
F: Future<Output = T>
```

允许函数接受不同类型的 Future，只要它们的输出类型符合 `T`。

### 两个竞争 Future 需要相同输出类型

`race` 返回 `T`，因此：

```rust
F1: Future<Output = T>
F2: Future<Output = T>
```

例如一个返回 `&str`、另一个返回 `usize` 的 Future 不能直接传给当前
`race` 实现。

### 超时不代表操作从未执行

超时只表示函数停止等待并取消尚未完成的 Future。它不能保证外部操作没有
产生部分效果。对于有副作用的操作，需要明确取消语义。

## 10. 核心要点

1. `tokio::select!` 在多个 Future 之间等待，先完成的分支决定结果。
2. `with_timeout` 用业务 Future 与计时器竞争。
3. `race` 返回两个同输出类型 Future 中先完成者的结果。
4. `sleep(...).await` 不会阻塞操作系统线程。
5. `select!` 结束后，尚未完成的分支 Future 默认会被丢弃。
6. 丢弃 Future 不会自动撤销已产生的外部副作用。
7. 单个 Future 的超时场景也可以使用 `tokio::time::timeout`。
8. 多个分支同时可完成时，不应依赖固定的获胜顺序。
