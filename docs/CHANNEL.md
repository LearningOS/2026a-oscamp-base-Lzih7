# 使用 Channel 进行线程通信

本练习位于 `exercises/01_concurrency_sync/03_channel/src/lib.rs`，使用 Rust 标准库的 `std::sync::mpsc` 在多个线程之间传递消息。

## 1. `mpsc` 的含义

`mpsc` 是 multiple producer, single consumer 的缩写，表示：

- 可以有多个生产者（`Sender<T>`）。
- 通常只有一个消费者（`Receiver<T>`）。
- 生产者向 channel 发送数据，消费者从 channel 接收数据。

创建 channel：

```rust
let (tx, rx) = mpsc::channel();
```

其中 `tx` 是发送端，`rx` 是接收端。channel 会根据发送的数据推断类型，例如这里可以是 `Sender<String>` 和 `Receiver<String>`。

## 2. 发送消息

发送端通过 `send()` 发送数据：

```rust
tx.send(item).unwrap();
```

`send()` 会取得消息的所有权。发送成功返回 `Ok(())`；如果接收端已经被释放，则返回错误。

在 `simple_send_recv` 中，`items` 被移动到生产线程，线程逐个发送其中的字符串：

```rust
let thread = thread::spawn(move || {
    for item in items {
        tx.send(item).unwrap();
    }
});
```

`move` 保证线程拥有 `items` 和 `tx` 的使用权，避免线程引用已经离开作用域的数据。

## 3. 接收消息

接收端可以使用 `recv()`：

```rust
while let Ok(item) = rx.recv() {
    result.push(item);
}
```

`recv()` 在暂时没有消息时会阻塞当前线程。当所有发送端都被释放后，`recv()` 返回 `Err`，循环结束。

也可以直接迭代接收端：

```rust
let result: Vec<String> = rx.into_iter().collect();
```

这个迭代器会持续等待消息，直到所有 `Sender` 都被丢弃。

## 4. 多个生产者

`Sender` 实现了 `Clone`，因此可以为每个生产线程创建独立的发送端：

```rust
for i in 0..n_producers {
    let tx = tx.clone();
    threads.push(thread::spawn(move || {
        tx.send(format!("msg from {}", i)).unwrap();
    }));
}
```

虽然发送端被克隆了多份，但这些发送端仍然连接到同一个 channel。每个线程发送的消息可能以任意顺序到达，因为线程调度顺序是不确定的。

## 5. 为什么必须释放原始发送端

循环中创建的克隆发送端在线程结束时会自动释放，但最初的 `tx` 仍然存在。因此接收端可能一直等待：

```rust
drop(tx);
let messages: Vec<String> = rx.into_iter().collect();
```

只有原始发送端和所有克隆发送端都被释放，接收迭代器才知道不会再有新消息，随后结束。

这是 channel 代码中非常重要的生命周期规则：接收结束不是由“当前没有消息”决定，而是由“所有发送端都已经关闭”决定。

## 6. `join()` 与消息接收的顺序

在生产线程发送消息时，主线程应同时接收消息。这样可以让生产和消费并行进行：

```rust
let messages: Vec<String> = rx.into_iter().collect();

for thread in threads {
    thread.join().unwrap();
}
```

接收迭代器会等待所有发送端关闭，之后再通过 `join()` 确认生产线程正常结束。如果线程发生 panic，`join()` 会返回错误，`unwrap()` 会使当前线程 panic。

## 7. 为什么需要排序

多个线程发送消息时，消息到达顺序不由线程编号决定。因此 `multi_producer` 在收集后排序：

```rust
messages.sort();
```

排序后，测试和调用者可以得到稳定的字典序结果。排序并不会改变 channel 的通信机制，只是对接收后的结果进行确定性处理。

## 8. 核心要点

1. `mpsc::channel()` 创建一个多生产者、单消费者 channel。
2. `Sender::send()` 发送数据，并转移消息所有权。
3. `Sender::clone()` 可以为多个线程创建生产者。
4. 所有 `Sender` 被释放后，接收端才会结束。
5. `recv()` 没有消息时会阻塞，接收迭代器会持续接收直到 channel 关闭。
6. `join()` 用于等待线程结束并检查线程是否发生 panic。
7. 多线程消息顺序不稳定，需要排序时应在接收完成后统一处理。
