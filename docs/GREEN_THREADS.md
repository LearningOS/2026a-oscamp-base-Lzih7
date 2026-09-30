# 协作式绿色线程调度器

本练习位于 `exercises/04_context_switch/02_green_threads/src/lib.rs`，在
RISC-V 上下文切换的基础上实现一个最小的协作式绿色线程调度器。

绿色线程不是由操作系统直接调度的内核线程，而是由用户态调度器管理的
执行任务。线程只有主动调用 `yield_now()`，或者执行完入口函数后，才会
把控制权交还给调度器。

## 1. 绿色线程与内核线程

传统内核线程由操作系统调度：

```text
操作系统保存线程上下文
操作系统选择下一个线程
操作系统恢复目标线程
```

绿色线程由库或运行时调度：

```text
绿色线程主动让出
用户态调度器选择下一个线程
用户态上下文切换恢复目标线程
```

本练习使用一个操作系统线程承载多个绿色线程。每个绿色线程拥有独立的
栈和寄存器上下文，但不需要创建独立的内核线程。

## 2. 线程状态

```rust
pub enum ThreadState {
    Ready,
    Running,
    Finished,
}
```

三种状态含义如下：

| 状态 | 含义 |
|---|---|
| `Ready` | 已创建，可以被调度运行 |
| `Running` | 当前正在运行 |
| `Finished` | 入口函数已经返回，不再参与调度 |

状态转换通常是：

```text
Ready -> Running
Running -> Ready       // 调用 yield_now()
Running -> Finished    // 入口函数返回
```

已经处于 `Finished` 的线程不会重新进入 `Ready`，因此不会被再次执行。

## 3. 绿色线程数据结构

```rust
struct GreenThread {
    ctx: TaskContext,
    state: ThreadState,
    _stack: Option<Vec<u8>>,
    entry: Option<extern "C" fn()>,
}
```

- `ctx`：保存该线程的 `sp`、`ra` 和 `s0-s11`。
- `state`：记录调度状态。
- `_stack`：持有线程栈的所有权，防止栈内存被释放。
- `entry`：线程第一次运行时需要调用的用户入口函数。

线程栈必须由 `GreenThread` 持有整个生命周期。若 `Vec<u8>` 被释放，
上下文中的 `sp` 就会指向无效内存。

## 4. 调度器结构

```rust
pub struct Scheduler {
    threads: Vec<GreenThread>,
    current: usize,
}
```

- `threads`：所有绿色线程，包括索引为 0 的主线程。
- `current`：当前正在运行的线程索引。

创建调度器时，主线程作为第一个线程加入：

```rust
state: ThreadState::Running,
_stack: None,
entry: None,
```

主线程使用操作系统提供的栈，因此不需要额外分配绿色线程栈。

## 5. 创建线程

`spawn` 的主要流程是：

```text
1. 分配独立栈
2. 创建空的 TaskContext
3. 设置 sp 为栈顶
4. 设置 ra 为 thread_wrapper
5. 保存用户入口函数
6. 将线程状态设置为 Ready
```

初始化上下文时，`ra` 不是直接设置为用户入口，而是设置为：

```rust
thread_wrapper as *const () as usize
```

这是因为第一次切换到新线程时，汇编代码会执行：

```asm
ret
```

`ret` 会跳转到 `ra`。统一进入 `thread_wrapper` 后，运行时才能在用户
入口返回时执行 `thread_finished()`，将线程标记为完成并切换回调度器。

## 6. 独立栈与栈对齐

```rust
fn alloc_stack() -> (Vec<u8>, usize) {
    let stack = Vec::with_capacity(STACK_SIZE);
    let stack_top = (stack.as_ptr() as usize + STACK_SIZE) & !15;
    (stack, stack_top)
}
```

RISC-V 栈向低地址增长，因此栈顶位于分配区域的高地址：

```text
低地址                         高地址
|--------------------------------|
^                                ^
栈底                             stack_top
```

栈顶通过 `& !15` 对齐到 16 字节边界，满足 RISC-V ABI 的要求。

`Vec::with_capacity` 分配了容量，但长度为 0。这里把内存作为原始栈空间
使用，不通过普通切片访问其中的元素。返回的 `Vec` 必须保存到线程结束，
所以放入 `_stack` 字段中。

## 7. `thread_wrapper`

```rust
extern "C" fn thread_wrapper() {
    let entry = unsafe { core::ptr::read(&raw const CURRENT_THREAD_ENTRY) };
    if let Some(f) = entry {
        unsafe { CURRENT_THREAD_ENTRY = None };
        f();
    }
    thread_finished();
}
```

它承担三个职责：

1. 从全局变量中取出当前线程的入口函数。
2. 调用用户提供的入口函数。
3. 入口函数返回后，通知调度器线程已经完成。

入口函数只应被取出并调用一次，因此调度器在切换到线程前会执行：

```rust
CURRENT_THREAD_ENTRY = self.threads[next_index].entry.take();
```

`take()` 将 `Option<fn()>` 替换为 `None`，防止同一个入口函数被重复调用。

## 8. `run` 调度循环

```rust
pub fn run(&mut self) {
    unsafe {
        SCHEDULER = self as *mut Scheduler;
    }

    while self.threads.iter().skip(1).any(|thread| {
        thread.state != ThreadState::Finished
    }) {
        self.schedule_next();
    }

    unsafe {
        SCHEDULER = std::ptr::null_mut();
        CURRENT_THREAD_ENTRY = None;
    }
}
```

运行过程如下：

```text
设置全局 SCHEDULER
    |
循环检查非主线程是否全部 Finished
    |
调用 schedule_next()
    |
绿色线程运行，并可能 yield
    |
恢复主线程后继续循环
    |
所有线程完成，清理全局状态
```

主线程被保留在 `threads[0]` 中，但结束条件只检查 `threads[1..]`，
因为主线程本身不属于被调度的绿色线程任务。

## 9. Round-robin 调度

调度器从当前线程之后开始查找下一个 `Ready` 线程：

```rust
let next_index = (1..=self.threads.len())
    .map(|offset| (old_index + offset) % self.threads.len())
    .find(|&index| self.threads[index].state == ThreadState::Ready);
```

例如线程排列为：

```text
0: main
1: A
2: B
3: C
```

如果当前线程是 `A`，查找顺序为：

```text
B -> C -> main -> A
```

这种策略称为 round-robin（轮转调度）。它不会一直选择同一个线程，
而是按照环形顺序给每个 Ready 线程运行机会。

找到下一个线程后：

```rust
if self.threads[old_index].state != ThreadState::Finished {
    self.threads[old_index].state = ThreadState::Ready;
}
self.threads[next_index].state = ThreadState::Running;
self.current = next_index;
```

如果旧线程只是主动让出，则回到 `Ready`；如果旧线程已经结束，则保持
`Finished`。

## 10. `yield_now` 的执行流程

绿色线程调用：

```rust
yield_now();
```

实际流程是：

```text
任务 A 调用 yield_now()
    |
调用 SCHEDULER.schedule_next()
    |
A 标记为 Ready
    |
选择任务 B
    |
保存 A 的上下文
    |
加载 B 的上下文
    |
ret 跳转到 B
```

当未来调度器再次恢复 A 时，A 会从 `yield_now()` 调用之后继续执行，
而不是重新执行入口函数。

## 11. 线程完成的执行流程

用户入口函数返回后，控制权回到 `thread_wrapper`：

```rust
f();
thread_finished();
```

`thread_finished` 首先将当前线程设置为 `Finished`：

```rust
sched.threads[sched.current].state = ThreadState::Finished;
```

然后调用 `schedule_next()`，切换到下一个 Ready 线程。已经完成的线程
不会再次被选中。

如果所有绿色线程都完成，调度器会找到处于 Ready 状态的主线程并切回，
使 `run()` 恢复执行并退出循环。

## 12. 上下文切换

调度器使用与上一练习相同的 RISC-V 裸汇编：

```asm
sd sp, 0(a0)
sd ra, 8(a0)
sd s0, 16(a0)
...
sd s11, 104(a0)

ld sp, 0(a1)
ld ra, 8(a1)
ld s0, 16(a1)
...
ld s11, 104(a1)
ret
```

按照 RISC-V C ABI：

```text
a0 = old 上下文
a1 = new 上下文
```

保存当前线程的寄存器后，加载目标线程的寄存器。最后的 `ret` 使用新
上下文中的 `ra` 继续执行：

```text
首次运行：ra = thread_wrapper
再次运行：ra = 上次 switch_context 返回位置
```

这就是“暂停后恢复”的核心机制。

## 13. 全局指针与 `unsafe`

```rust
static mut SCHEDULER: *mut Scheduler = std::ptr::null_mut();
static mut CURRENT_THREAD_ENTRY: Option<extern "C" fn()> = None;
```

`yield_now()` 没有接收调度器参数，因此通过全局指针找到当前调度器：

```rust
unsafe {
    if !SCHEDULER.is_null() {
        (*SCHEDULER).schedule_next();
    }
}
```

这种设计简化了示例，但需要调用者保证：

1. `SCHEDULER` 只在 `run()` 执行期间有效。
2. `run()` 期间不会并行使用同一个调度器。
3. 调度器和其中的线程不会在上下文切换期间移动。
4. `CURRENT_THREAD_ENTRY` 只被当前调度流程访问。

因此访问这些全局状态和执行裸汇编都必须使用 `unsafe`。生产级运行时
通常会使用更严格的封装，避免裸的全局可变状态。

## 14. 测试执行顺序

假设有两个任务：

```text
任务 A：+1，yield，+10，yield，+100
任务 B：+1，yield，+10
```

调度顺序大致为：

```text
main -> A -> B -> main -> A -> B -> main -> A -> main
```

执行结果：

```text
A: 1 + 10 + 100 = 111
B: 1 + 10 = 11
总和 = 122
```

这个测试同时验证了：

- 线程能够首次启动。
- `yield_now()` 能暂停并恢复线程。
- 多个线程按照轮转顺序运行。
- 线程结束后不会再次运行。
- 最终能回到主线程。

## 15. 协作式调度的限制

协作式绿色线程必须主动让出控制权：

```rust
yield_now();
```

如果某个线程执行无限循环且从不调用 `yield_now()`，其他线程将无法运行。
这与抢占式调度不同，抢占式调度由定时器中断强制切换线程。

因此协作式调度适合：

- 用户态协程。
- 明确控制执行权的任务。
- 需要低成本切换的运行时。

它不适合运行可能长期占用 CPU 且不主动让出的不可信代码。

## 16. 平台限制

文件使用：

```rust
#![cfg(target_arch = "riscv64")]
```

因此只有 RISC-V 目标会编译实际实现。x86_64 主机上执行普通
`cargo test` 时，测试模块会被条件编译排除，不能验证上下文切换。

实际验证需要：

```bash
cargo test \
  --manifest-path exercises/04_context_switch/02_green_threads/Cargo.toml \
  --target riscv64gc-unknown-linux-gnu
```

同时必须配置 `riscv64-linux-gnu-gcc` 和 QEMU 运行环境。

## 17. 核心要点

1. 绿色线程由用户态调度器管理，不是独立内核线程。
2. 每个线程拥有独立栈、上下文和入口函数。
3. `Ready`、`Running`、`Finished` 描述线程生命周期。
4. `thread_wrapper` 负责调用入口并处理线程完成。
5. `yield_now()` 触发协作式上下文切换。
6. `schedule_next()` 使用 round-robin 选择下一个 Ready 线程。
7. `entry.take()` 确保入口函数只执行一次。
8. `_stack` 保持线程栈存活，防止 `sp` 悬空。
9. 上下文切换通过保存和恢复 `sp`、`ra`、`s0-s11` 实现。
10. 协作式调度要求任务主动让出 CPU。
