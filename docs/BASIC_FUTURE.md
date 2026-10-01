# 手写 Future

本练习位于 `exercises/05_async_programming/01_basic_future/src/lib.rs`，通过
手动实现 `Future` trait 理解 Rust 异步编程的核心机制。

Rust 的 `async/await` 本质上会被编译器转换成状态机，而 `Future` trait
就是这个状态机对外暴露的接口。本练习没有使用 `async fn`，而是直接手写
两个 Future：

- `CountDown`：每次被轮询时倒计时一次，归零后返回 `"liftoff!"`。
- `YieldOnce`：第一次轮询让出一次，第二次轮询完成。

## 1. `Future` trait

标准库中的 `Future` trait 核心形式是：

```rust
pub trait Future {
    type Output;

    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}
```

它包含两个关键部分：

```rust
type Output;
```

表示 Future 完成后产生的值。

```rust
fn poll(...) -> Poll<Self::Output>;
```

表示运行时尝试推动这个 Future 向前执行一步。

Future 不是主动运行的。它需要由异步运行时反复调用 `poll`，直到返回
`Poll::Ready`。

## 2. `Poll`

`poll` 的返回值是：

```rust
Poll<Self::Output>
```

它有两种状态：

```rust
Poll::Pending
Poll::Ready(value)
```

含义如下：

| 返回值 | 含义 |
|---|---|
| `Poll::Pending` | 当前还没有完成，之后需要再次 poll |
| `Poll::Ready(value)` | Future 已完成，返回最终结果 |

可以理解为：

```text
Pending：我现在还没好，之后再来问我
Ready：我已经完成了，这是结果
```

## 3. `Context` 与 `Waker`

`poll` 的第二个参数是：

```rust
cx: &mut Context<'_>
```

`Context` 中最重要的是 `Waker`：

```rust
cx.waker()
```

`Waker` 用来通知异步运行时：

```text
这个 Future 之后可能可以继续推进了，请再次 poll 我
```

在本练习中，Future 自己调用：

```rust
cx.waker().wake_by_ref();
```

表示当前 Future 改变了内部状态，希望运行时稍后再次调用 `poll`。

如果一个 Future 返回 `Pending`，但永远不唤醒 Waker，运行时可能就不会再
主动轮询它，Future 也就无法完成。

## 4. `Pin<&mut Self>`

`poll` 的第一个参数是：

```rust
self: Pin<&mut Self>
```

它看起来比普通方法的 `&mut self` 复杂。原因是：有些 Future 内部可能保存
指向自身字段的引用，这类结构一旦被移动，内部引用就可能失效。

`Pin` 的作用是限制被包裹对象的位置移动。对于本练习的两个结构体来说：

```rust
pub struct CountDown {
    pub count: u32,
}

pub struct YieldOnce {
    yielded: bool,
}
```

它们都没有自引用字段，因此可以安全地用：

```rust
let this = self.get_mut();
```

从 `Pin<&mut Self>` 取出普通的 `&mut Self`。

需要注意：`get_mut()` 会消耗 `self` 这个 `Pin<&mut Self>` 值，所以通常
应该只调用一次：

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
    let this = self.get_mut();
    // 后面都使用 this
}
```

不要写成：

```rust
let count = self.get_mut().count;
self.get_mut().count -= 1;
```

因为第一次 `self.get_mut()` 已经把 `self` move 掉，第二次再用会产生
`use of moved value: self` 编译错误。

## 5. `CountDown` 的状态机

`CountDown` 保存一个计数器：

```rust
pub struct CountDown {
    pub count: u32,
}
```

它的 `Output` 是：

```rust
type Output = &'static str;
```

也就是说，Future 完成后返回一个静态字符串。

实现如下：

```rust
impl Future for CountDown {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.get_mut();
        if this.count == 0 {
            Poll::Ready("liftoff!")
        } else {
            this.count -= 1;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}
```

执行逻辑是：

```text
count == 0：
    返回 Ready("liftoff!")

count > 0：
    count 减 1
    唤醒 Waker
    返回 Pending
```

例如：

```rust
CountDown::new(3).await
```

大致经历：

```text
第 1 次 poll：count = 3 -> 2，Pending
第 2 次 poll：count = 2 -> 1，Pending
第 3 次 poll：count = 1 -> 0，Pending
第 4 次 poll：count = 0，Ready("liftoff!")
```

所以 `CountDown::new(3)` 并不是 poll 三次就返回，而是前三次递减，第四次
观察到 `count == 0` 才完成。

## 6. `YieldOnce` 的状态机

`YieldOnce` 只保存一个布尔状态：

```rust
pub struct YieldOnce {
    yielded: bool,
}
```

初始状态：

```rust
yielded = false
```

实现如下：

```rust
impl Future for YieldOnce {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.get_mut();
        if this.yielded {
            Poll::Ready(())
        } else {
            this.yielded = true;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}
```

执行过程是：

```text
第 1 次 poll：
    yielded = false
    修改为 true
    唤醒 Waker
    返回 Pending

第 2 次 poll：
    yielded = true
    返回 Ready(())
```

它是最小的异步状态机示例：第一次调用时主动让出，第二次调用时完成。

## 7. `await` 背后发生了什么

测试中有：

```rust
let result = CountDown::new(3).await;
```

`await` 并不是阻塞当前操作系统线程等待结果，而是让异步运行时不断推动
Future：

```text
poll future
    |
如果 Pending：
    当前任务让出执行权
    等待 Waker 通知
    之后再次 poll
    |
如果 Ready：
    取出返回值，继续向下执行
```

在本练习里使用的是 `tokio::test`，所以 Tokio 运行时负责创建任务、调用
`poll`、处理 Waker，并在 Future 完成后继续执行测试断言。

## 8. 为什么返回 Pending 前要唤醒

以 `CountDown` 为例：

```rust
this.count -= 1;
cx.waker().wake_by_ref();
Poll::Pending
```

这里虽然还没完成，但内部状态已经变化了。调用 `wake_by_ref()` 是在告诉
运行时：

```text
我已经向完成状态推进了一步，请继续调度我
```

如果省略唤醒：

```rust
this.count -= 1;
Poll::Pending
```

运行时可能认为这个 Future 还在等待外部事件，例如网络、定时器或 IO，
于是不会马上再次 poll。这样测试就可能卡住。

## 9. `wake_by_ref` 与 `wake`

本练习使用：

```rust
cx.waker().wake_by_ref();
```

它通过引用唤醒任务，不会消耗 `Waker` 本身。

还有一种方法是：

```rust
cx.waker().clone().wake();
```

`wake()` 会消耗 Waker，因此常见写法需要先 `clone()`。在这里只是通知运行时
重新 poll 当前任务，用 `wake_by_ref()` 更直接。

## 10. 与同步函数的区别

同步倒计时可能写成：

```rust
while count > 0 {
    count -= 1;
}
return "liftoff!";
```

它会在一次函数调用中直接跑完。

Future 的方式是：

```text
每次 poll 只推进一步
没有完成就返回 Pending
完成时返回 Ready
```

这种设计允许运行时在多个异步任务之间切换，而不是让一个任务长期占用执行权。

## 11. 核心要点

1. `Future` 是异步状态机的统一接口。
2. `poll` 推动 Future 向前执行一步。
3. `Poll::Pending` 表示尚未完成，`Poll::Ready` 表示完成。
4. `Waker` 用于通知运行时再次 poll。
5. `Pin<&mut Self>` 防止某些 Future 在 poll 过程中被移动。
6. 对本练习这种非自引用类型，可以用 `self.get_mut()` 获取 `&mut Self`。
7. `get_mut()` 会消耗 `self`，所以应先保存为 `let this = self.get_mut();`。
8. `CountDown` 用 `count` 表示状态。
9. `YieldOnce` 用 `yielded` 表示是否已经让出过一次。
10. `await` 背后就是运行时反复 poll Future，直到它返回 Ready。
