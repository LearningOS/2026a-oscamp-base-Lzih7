# 栈式协程与 RISC-V 上下文切换

本练习位于 `exercises/04_context_switch/01_stack_coroutine/src/lib.rs`，使用
RISC-V 64 位内联汇编实现最小的上下文切换。上下文切换是操作系统线程调度、
用户态协程和绿色线程的基础。

## 1. 什么是上下文

上下文就是一个任务暂停后，未来能够继续执行所需的处理器状态。本练习保存：

```text
sp       栈指针
ra       返回地址
s0-s11   RISC-V callee-saved 寄存器
```

这些寄存器共 14 个，每个寄存器 8 字节，共 112 字节。

```rust
#[repr(C)]
pub struct TaskContext {
    pub sp: u64,   // offset 0
    pub ra: u64,   // offset 8
    pub s0: u64,   // offset 16
    // ...
    pub s11: u64,  // offset 104
}
```

`#[repr(C)]` 保证字段按照 C ABI 顺序布局，使汇编中的固定偏移与 Rust
结构体布局一致。如果改变字段顺序或类型，必须同步修改汇编偏移。

## 2. 为什么只保存 callee-saved 寄存器

RISC-V ABI 将寄存器分为两类：

```text
caller-saved：调用者负责保存
callee-saved：被调用者负责保存
```

本练习的切换点按照函数调用边界处理，因此保存 `sp`、`ra` 和 `s0-s11`
就足以让被切换任务恢复到正确的执行状态。完整的抢占式内核上下文通常还需要
保存更多寄存器，例如 `a0-a7`、`t0-t6` 以及浮点寄存器。

## 3. 初始化任务上下文

```rust
pub fn init(&mut self, stack_top: usize, entry: usize) {
    self.ra = entry as u64;
    self.sp = (stack_top & !15) as u64;
}
```

第一次切换到一个新任务时，它还没有被真正运行过，因此没有需要恢复的旧
调用现场。初始化时设置：

- `ra = entry`：切换汇编最后执行 `ret`，跳转到入口函数。
- `sp = stack_top`：使用新任务自己的栈。
- `s0-s11 = 0`：由 `empty()` 初始化，切换时正常加载。

RISC-V ABI 要求函数入口的栈满足 16 字节对齐，所以使用：

```rust
stack_top & !15
```

清除地址低四位，使栈顶向下对齐到 16 字节边界。

## 4. 分配协程栈

```rust
pub fn alloc_stack() -> (Vec<u8>, usize) {
    let buf = Vec::with_capacity(STACK_SIZE);
    let stack_top = (buf.as_ptr() as usize + STACK_SIZE) & !15;
    (buf, stack_top)
}
```

RISC-V 栈向低地址增长，因此：

```text
buf.as_ptr()                 低地址，栈底
buf.as_ptr() + STACK_SIZE    高地址，栈顶
```

返回 `Vec<u8>` 是为了保持这块堆内存的所有权和生命周期。调用者必须让
`Vec` 一直存活到协程结束；如果缓冲区被释放，`sp` 就会指向无效内存。

`Vec::with_capacity` 分配容量但长度仍为 0。这对本练习是可行的，因为汇编
只把这块内存作为栈空间使用；如果需要以普通 Rust 切片访问内容，则必须
另外初始化长度。

## 5. 调用约定与 `switch_context`

```rust
#[unsafe(naked)]
pub unsafe extern "C" fn switch_context(
    _old: &mut TaskContext,
    _new: &TaskContext,
)
```

按照 RISC-V C ABI：

```text
a0 = old 上下文地址
a1 = new 上下文地址
```

函数使用 `#[unsafe(naked)]`，禁止编译器生成普通函数序言和尾声。因为切换
函数会直接修改 `sp` 和 `ra`，自动生成的 prologue/epilogue 可能使用错误的
栈或破坏刚刚恢复的寄存器。

## 6. 保存旧上下文

汇编首先把当前任务的寄存器写入 `old`：

```asm
sd sp, 0(a0)
sd ra, 8(a0)
sd s0, 16(a0)
...
sd s11, 104(a0)
```

`sd` 是 store doubleword，将 64 位寄存器写入内存。保存的是“当前任务被
切出时”的处理器状态。

特别是 `ra` 保存当前函数调用返回的位置。未来恢复这个上下文时，`ret`
会从保存的 `ra` 继续执行。

## 7. 恢复新上下文

随后从 `new` 读取目标任务状态：

```asm
ld sp, 0(a1)
ld ra, 8(a1)
ld s0, 16(a1)
...
ld s11, 104(a1)
```

`ld` 是 load doubleword，将内存中的 64 位值加载到寄存器。

恢复完成后：

```asm
li a0, 0
li a1, 0
ret
```

清零参数寄存器可以避免新任务继续看到上下文指针。最后的 `ret` 等价于：

```asm
jalr zero, 0(ra)
```

它跳转到新上下文中的 `ra`。对于第一次运行的协程，这个地址就是
`TaskContext::init` 设置的入口函数；对于已经运行过的协程，则回到它上次
被切出的指令位置。

## 8. 一次切换的完整流程

假设当前任务是 `main`，目标任务是 `task`：

```text
1. a0 指向 main_ctx，a1 指向 task_ctx
2. 保存当前 sp、ra、s0-s11 到 main_ctx
3. 从 task_ctx 加载 sp、ra、s0-s11
4. ret 跳转到 task_ctx.ra
5. task 使用自己的栈运行
6. task 再次 switch_context 时，保存 task_ctx
7. 加载 main_ctx
8. ret 回到 main 上次 switch_context 返回的位置
```

关键点是：`switch_context` 并不返回一个普通的返回值，它通过恢复另一组
寄存器和执行 `ret` 把控制权交给另一个上下文。将来恢复旧上下文时，原先的
函数调用才会从切换点继续向下执行。

## 9. 为什么函数是 `unsafe`

上下文切换包含多个 Rust 无法自动验证的条件：

1. `old` 和 `new` 必须指向有效的 `TaskContext`。
2. 两个上下文的内存布局必须与汇编偏移一致。
3. `new.sp` 必须指向有效且对齐的栈。
4. `new.ra` 必须是有效的代码地址。
5. 栈缓冲区必须在整个任务生命周期内保持存活。
6. 切换期间不能违反 Rust 引用的别名和生命周期规则。

因此接口标记为 `unsafe`，调用者必须保证这些前置条件。

## 10. `unsafe` 与裸汇编的边界

这段代码使用：

```rust
core::arch::naked_asm!(...)
```

裸汇编不会自动理解 Rust 类型、生命周期或借用规则。Rust 代码负责组织
`TaskContext` 和栈的生命周期，汇编代码负责按 ABI 读写寄存器。两者之间的
契约包括：

```text
Rust 字段顺序 == 汇编偏移
Rust 参数顺序 == a0/a1
Rust 栈地址 == 汇编恢复的 sp
Rust 入口地址 == 汇编 ret 使用的 ra
```

任何一项不一致，都可能导致崩溃、跳转到错误地址或静默破坏状态。

## 11. 与普通函数调用的区别

普通函数调用由编译器自动处理：

```text
调用者准备参数
被调用者建立栈帧
函数保存必要寄存器
函数返回并恢复调用者状态
```

上下文切换则手动完成这些工作，而且目标不是返回原调用者，而是切换到
另一个任务的栈和返回地址。因此它是操作系统调度器中最底层、最依赖 ABI
的部分之一。

## 12. 测试的含义

### `test_alloc_stack`

验证返回的栈顶位于分配区域高地址，并满足 16 字节对齐。

### `test_context_init`

验证 `init` 将入口地址写入 `ra`，并设置有效栈指针。

### `test_switch_to_task`

测试流程为：

```text
main 上下文切换到 task
task_entry 修改 COUNTER
task 切换回 main
main 检查 COUNTER
```

这同时验证了入口跳转、协程栈使用、上下文保存以及返回原执行点。

## 13. 平台限制

文件顶部有：

```rust
#![cfg(target_arch = "riscv64")]
```

因此该 crate 只在 `riscv64` 目标上编译实际实现。x86_64 主机不能直接执行
RISC-V 汇编，通常需要使用项目提供的 QEMU 或交叉编译流程验证。

## 14. 核心要点

1. 上下文是任务恢复所需的寄存器和栈状态。
2. `TaskContext` 必须使用 `#[repr(C)]`，并与汇编偏移严格匹配。
3. `sp`、`ra` 和 `s0-s11` 是本练习保存的 callee-saved 状态。
4. 新任务的 `ra` 指向入口函数，`ret` 负责第一次跳转。
5. 协程栈向下增长，栈顶必须满足 16 字节对齐。
6. `#[unsafe(naked)]` 防止编译器生成会干扰切换的序言和尾声。
7. `sd` 保存寄存器，`ld` 恢复寄存器。
8. `Vec<u8>` 必须保持存活，否则上下文中的栈指针会悬空。
9. 上下文切换依赖 RISC-V ABI，只能在对应架构上运行。
