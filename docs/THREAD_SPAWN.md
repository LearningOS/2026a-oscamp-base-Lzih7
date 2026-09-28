# 线程创建与基础并发

本练习位于 `exercises/01_concurrency_sync/01_thread_spawn/src/lib.rs`，重点学习 Rust 标准库中的线程创建、数据所有权转移、线程结果回收以及几种常见的线程操作。

## 1. 创建线程与移动所有权

`thread::spawn` 接收一个闭包，并在新的操作系统线程中执行：

```rust
let handle = thread::spawn(move || {
    numbers.into_iter().map(|n| n * 2).collect::<Vec<_>>()
});
```

这里的 `move` 会把 `numbers` 的所有权移动到闭包中。这样新线程不依赖创建它的函数栈帧，即使外层函数已经返回，线程仍然可以安全地使用这些数据。

使用 `move` 后，原线程不能再使用被移动的变量。对于 `Vec<T>`，`into_iter()` 会取得 vector 中元素的所有权，并逐个消费元素。

## 2. 使用 `JoinHandle` 等待线程

`thread::spawn` 返回 `JoinHandle<T>`，其中 `T` 是线程闭包的返回值。调用：

```rust
let result = handle.join().unwrap();
```

会等待线程执行结束，并取回线程返回的结果。`join()` 的返回类型是：

```rust
Result<T, Box<dyn Any + Send>>
```

如果线程正常结束，得到 `Ok(T)`；如果线程发生 panic，得到 `Err`。练习中使用 `unwrap()` 是因为这些线程在预期路径上不会 panic。

## 3. 并行计算

`parallel_sum` 为两个 vector 分别创建线程：

```rust
let handle_a = thread::spawn(move || a.into_iter().sum::<i32>());
let handle_b = thread::spawn(move || b.into_iter().sum::<i32>());

(handle_a.join().unwrap(), handle_b.join().unwrap())
```

两个求和任务可以同时运行。每个 vector 被移动到对应线程中，因此两个线程之间没有共享可变数据，不需要额外的锁。

## 4. 命名线程与休眠

`thread::Builder` 可以设置线程名称，然后创建线程：

```rust
thread::Builder::new()
    .name("sleeper".to_owned())
    .spawn(move || {
        thread::sleep(Duration::from_millis(ms));
        value
    })
```

线程名称主要用于调试和日志定位。`thread::sleep` 只会阻塞当前线程，不会阻塞其他线程。

## 5. 线程局部存储

`thread_local!` 用于声明每个线程独立拥有一份数据：

```rust
thread_local! {
    static THREAD_COUNT: RefCell<usize> = RefCell::new(0);
}
```

通过 `.with()` 访问线程自己的副本：

```rust
THREAD_COUNT.with(|cell| {
    let mut count = cell.borrow_mut();
    *count += 1;
    *count
})
```

不同线程调用 `increment_thread_local` 时，计数器互不影响。例如每个线程第一次调用都返回 `1`，第二次调用都返回 `2`。

`RefCell` 提供运行时借用检查。这里使用 `borrow_mut()` 获取可变借用，并在闭包结束前更新计数。

## 6. 作用域线程

普通的 `thread::spawn` 通常要求闭包满足 `'static`，因为线程可能在外层函数返回后继续运行。`thread::scope` 保证作用域结束前所有线程都已经完成，因此线程可以借用栈上的切片：

```rust
thread::scope(|scope| {
    let handle_a = scope.spawn(|| a.iter().sum::<i32>());
    let handle_b = scope.spawn(|| b.iter().sum::<i32>());

    (handle_a.join().unwrap(), handle_b.join().unwrap())
})
```

这里没有移动 `a` 和 `b` 的所有权，线程只是临时借用它们。作用域结束后，原切片仍然可以继续使用。

## 7. 处理线程 panic

线程 panic 不会直接以普通返回值传递给调用者，而是由 `join()` 返回错误：

```rust
thread::spawn(move || {
    if should_panic {
        panic!("oops");
    }
    value
})
.join()
.map_err(|_| ())
```

当线程返回 `value` 时，`join()` 得到 `Ok(value)`；当线程 panic 时，得到 `Err(...)`，`map_err(|_| ())` 将错误类型统一转换为 `()`，因此函数接口变为 `Result<i32, ()>`。

## 8. 核心要点

1. `thread::spawn` 创建线程，`move` 将数据所有权转移给线程。
2. `JoinHandle::join` 等待线程结束并获取返回值。
3. 独立数据可以直接分发给多个线程进行并行计算。
4. `thread_local!` 为每个线程提供独立的数据副本。
5. `thread::scope` 允许线程安全地借用非 `'static` 数据。
6. 线程 panic 应通过 `join()` 的 `Result` 进行处理。
