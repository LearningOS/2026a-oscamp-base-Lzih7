# Tokio 异步 Channel

本练习位于 `exercises/05_async_programming/03_async_channel/src/lib.rs`，使用
`tokio::sync::mpsc` 实现异步生产者消费者模型。

`mpsc` 表示 multiple producer, single consumer，即多个生产者、单个消费者。
生产者通过 `Sender` 发送消息，消费者通过 `Receiver` 接收消息。Tokio 的
异步 channel 可以在任务之间传递数据，并在缓冲区满或没有消息时通过
`.await` 让出执行权。

## 1. 创建异步通道

代码使用：

```rust
let (tx, mut rx) = mpsc::channel(items.len().max(1));
```

`mpsc::channel(capacity)` 会创建一个有界异步通道：

```text
tx：发送端 Sender
rx：接收端 Receiver
capacity：通道缓冲区容量
```

容量表示通道中最多可以暂存多少条消息。发送速度快于接收速度时，消息会先
进入缓冲区；缓冲区满后，`send().await` 会等待接收端消费消息。

这里使用：

```rust
items.len().max(1)
```

保证容量至少为 1。Tokio 的 bounded channel 容量不能为 0，因此空输入时也
需要创建一个容量为 1 的通道。

## 2. `tx.send(...).await`

生产者发送消息：

```rust
tx.send(item).await.unwrap();
```

`send` 是异步操作，返回一个 Future。调用 `.await` 时：

```text
如果通道有空位：
    立即发送成功
如果通道已满：
    当前任务让出执行权，等待通道出现空位
如果接收端已关闭：
    返回错误
```

`unwrap()` 表示测试中假定接收端仍然存在。如果接收端提前被丢弃，`send`
会返回错误，`unwrap()` 会触发 panic。

## 3. `rx.recv().await`

消费者接收消息：

```rust
while let Some(item) = rx.recv().await {
    results.push(item);
}
```

`recv()` 也是异步操作。它的返回值是：

```rust
Option<T>
```

含义如下：

| 返回值 | 含义 |
|---|---|
| `Some(item)` | 成功收到一条消息 |
| `None` | 所有发送端都已关闭，且通道中没有剩余消息 |

因此：

```rust
while let Some(item) = rx.recv().await
```

表示持续接收消息，直到通道关闭。

需要注意，`recv()` 需要修改接收端内部状态，所以 `rx` 必须声明为可变：

```rust
let (tx, mut rx) = mpsc::channel(...);
```

## 4. 单生产者消费者模型

`producer_consumer` 的目标是：

```text
生产者依次发送 items 中的字符串
消费者接收所有字符串并按接收顺序返回 Vec
```

实现流程：

```rust
let (tx, mut rx) = mpsc::channel(items.len().max(1));
let mut results = Vec::new();
```

创建通道和结果数组。

生产者任务：

```rust
let producer = tokio::spawn(async move {
    for item in items {
        tx.send(item).await.unwrap();
    }
});
```

这里使用 `async move`，把 `items` 和 `tx` 移动进生产者任务。生产者遍历
`items`，逐条发送。

消费者任务：

```rust
let consumer = tokio::spawn(async move {
    while let Some(item) = rx.recv().await {
        results.push(item);
    }
    results
});
```

消费者持有唯一的 `rx`，不断接收消息。生产者任务结束后，`tx` 被释放；
当通道中剩余消息全部被消费后，`recv().await` 返回 `None`，循环结束。

最后等待任务完成：

```rust
producer.await.unwrap();
consumer.await.unwrap()
```

先等待生产者完成，再等待消费者返回收集结果。

## 5. 通道关闭机制

Tokio mpsc 通道关闭的关键条件是：

```text
所有 Sender 都被 drop
通道缓冲区中的消息全部被接收
```

满足这两个条件后：

```rust
rx.recv().await
```

会返回：

```rust
None
```

这也是消费者循环能够退出的原因。

如果仍然存在某个 `Sender`，即使它不会再发送消息，接收端也会认为通道仍
可能收到新消息，因此继续等待。

## 6. Fan-in 多生产者模式

`fan_in` 的目标是：

```text
创建多个生产者
每个生产者发送一条消息
一个消费者收集所有消息
排序后返回
```

核心代码：

```rust
let (tx, mut rx) = mpsc::channel(n_producers.max(1));
let mut producers = Vec::new();

for id in 0..n_producers {
    let tx = tx.clone();
    producers.push(tokio::spawn(async move {
        tx.send(format!("producer {id}: message")).await.unwrap();
    }));
}
```

每个生产者都需要自己的发送端，因此使用：

```rust
let tx = tx.clone();
```

`Sender` 可以克隆。所有克隆出来的 `Sender` 都指向同一个通道。

## 7. 为什么必须 `drop(tx)`

创建所有生产者后，需要显式释放原始发送端：

```rust
drop(tx);
```

原因是 `tx.clone()` 会为每个生产者创建新的发送端，但原始的 `tx` 仍然
存在于 `fan_in` 函数中。

如果不执行 `drop(tx)`：

```text
生产者任务中的 Sender 都结束并释放
原始 tx 仍然存在
接收端认为通道还没有关闭
rx.recv().await 继续等待
consumer 无法结束
fan_in 无法返回
```

这类问题在 mpsc 多生产者模型中很常见。只要接收端通过循环等待通道关闭，
就必须确保所有不再使用的 `Sender` 被释放。

## 8. 消费者收集与排序

消费者任务：

```rust
let consumer = tokio::spawn(async move {
    let mut results = Vec::new();
    while let Some(message) = rx.recv().await {
        results.push(message);
    }
    results.sort();
    results
});
```

多个生产者是并发执行的，因此消息到达顺序不固定。排序可以保证最终结果稳定：

```rust
vec![
    "producer 0: message",
    "producer 1: message",
    "producer 2: message",
]
```

排序发生在通道关闭之后，说明消费者已经收到所有生产者发送的消息。

## 9. 等待生产者和消费者

代码先等待所有生产者结束：

```rust
for producer in producers {
    producer.await.unwrap();
}
```

随后等待消费者返回结果：

```rust
consumer.await.unwrap()
```

生产者完成表示所有发送任务已经结束。由于原始 `tx` 已经被 `drop`，所有
发送端都释放后，消费者最终会收到 `None` 并返回结果。

## 10. `async move` 的作用

生产者任务使用：

```rust
tokio::spawn(async move {
    tx.send(...).await.unwrap();
});
```

`move` 表示把 `tx` 和 `id` 移动进异步任务中。Tokio 任务可能在当前函数的
后续时间点执行，因此任务必须拥有自己需要的数据，不能依赖局部借用。

对于字符串消息：

```rust
format!("producer {id}: message")
```

会在每个任务内部创建独立的 `String`，随后通过 channel 移动给消费者。

## 11. 核心要点

1. `tokio::sync::mpsc::channel` 创建有界异步通道。
2. `Sender` 可以克隆，支持多个生产者。
3. `Receiver` 通常只有一个，且 `recv()` 需要 `mut rx`。
4. `send().await` 在通道满时让出执行权。
5. `recv().await` 在没有消息时等待。
6. 所有 `Sender` 释放后，接收端最终得到 `None`。
7. 多生产者 fan-in 模式必须显式释放原始 `tx`。
8. 多生产者消息到达顺序不固定，排序可保证测试结果稳定。
