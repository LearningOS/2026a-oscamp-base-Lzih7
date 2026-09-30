# Rust 中的 `*` 与 `&`

Rust 中的 `*` 和 `&` 经常同时出现在指针、引用和借用操作中。例如，
`RwLock` 练习中的代码：

```rust
unsafe { &*self.lock.data.get() }
```

这行代码的作用是：从 `UnsafeCell<T>` 取得一个裸指针，将裸指针解引用，
再把它借用为 `&T` 返回。

## 1. `&` 的基本含义：创建引用

对一个值使用 `&`，表示借用这个值并创建共享引用：

```rust
let value = 42;
let reference: &i32 = &value;
```

这里：

```text
value       类型是 i32
&value      类型是 &i32
```

引用并不拥有数据，只是指向已有数据的安全访问入口。借用期间，原变量
仍然拥有数据。

```rust
let text = String::from("hello");
let view = &text;

println!("{}", text); // 仍然可以使用 text
println!("{}", view);
```

共享引用 `&T` 默认只能读取数据，不能通过它修改数据。

## 2. `&mut`：创建可变引用

使用 `&mut` 可以创建可变引用：

```rust
let mut value = 10;
let reference: &mut i32 = &mut value;
*reference += 1;
```

`&mut T` 允许修改被借用的数据，但 Rust 要求同一时刻不能存在冲突的借用：

```text
可以同时存在多个 &T
同一时刻只能存在一个 &mut T
&T 与 &mut T 不能同时指向同一数据
```

这正是 Rust 防止数据竞争的主要机制之一。

## 3. `*` 的第一种含义：解引用

如果变量保存的是引用或指针，使用 `*` 可以访问它指向的数据：

```rust
let value = 42;
let reference = &value;

assert_eq!(*reference, 42);
```

类型变化如下：

```text
reference       类型是 &i32
*reference      类型是 i32 位置上的值
```

对可变引用解引用后可以修改数据：

```rust
let mut value = 42;
let reference = &mut value;

*reference = 100;
assert_eq!(value, 100);
```

这里的 `*reference` 不是复制出一个独立对象，而是访问引用所指向的
原始数据位置。

## 4. `*` 的第二种含义：裸指针解引用

Rust 的裸指针类型包括：

```rust
*const T // 指向 T 的只读裸指针
*mut T   // 指向 T 的可变裸指针
```

例如：

```rust
let value = 42;
let pointer: *const i32 = &value;
```

裸指针可以为空、未对齐或指向无效地址，因此 Rust 不会自动保证它的安全性。
解引用裸指针必须放在 `unsafe` 块中：

```rust
let value = 42;
let pointer: *const i32 = &value;

let result = unsafe { *pointer };
assert_eq!(result, 42);
```

这与普通引用不同：

```rust
&T      // Rust 保证非空、有效且满足借用规则
*const T / *mut T // 这些安全条件由程序员负责
```

## 5. `&*ptr` 的含义

下面的写法经常用于把裸指针转换为引用：

```rust
unsafe { &*ptr }
```

它应该从右到左理解：

```rust
*ptr    // 解引用 ptr，访问它指向的 T
&(...)  // 借用这个 T，创建 &T
```

因此：

```rust
&*ptr
```

不是先创建引用再解引用，而是：

```text
裸指针 *mut T
    -> 解引用为数据位置
    -> 借用为共享引用 &T
```

完整展开形式是：

```rust
unsafe {
    let ptr: *mut T = ...;
    let reference: &T = &*ptr;
    reference
}
```

如果需要创建可变引用，则使用：

```rust
unsafe { &mut *ptr }
```

## 6. 当前 `RwLock` 代码的分析

`UnsafeCell<T>` 的 `get()` 方法返回裸指针：

```rust
let ptr: *mut T = self.lock.data.get();
```

读 Guard 的 `Deref` 需要返回 `&T`：

```rust
impl<T> Deref for RwLockReadGuard<'_, T> {
    type Target = T;

    fn deref(&self) -> &T {
        unsafe { &*self.lock.data.get() }
    }
}
```

这行代码可展开为：

```rust
fn deref(&self) -> &T {
    let ptr: *mut T = self.lock.data.get();
    unsafe { &*ptr }
}
```

写 Guard 的 `DerefMut` 类似，但返回可变引用：

```rust
fn deref_mut(&mut self) -> &mut T {
    unsafe { &mut *self.lock.data.get() }
}
```

代码能够安全的前提是：

1. 调用 `Deref` 时已经成功获得读锁。
2. 读锁保证没有写者同时修改数据。
3. 调用 `DerefMut` 时已经成功获得写锁。
4. 写锁保证当前线程独占访问数据。
5. `data` 在整个 Guard 生命周期内保持有效。

`UnsafeCell` 只允许内部可变性，不会自动提供这些同步保证；真正的保证
来自 `RwLock` 的原子状态和 Guard 生命周期。

## 7. `self` 为什么可以直接访问字段

在下面的表达式中：

```rust
self.lock.data.get()
```

`self` 的类型是：

```rust
&RwLockReadGuard<T>
```

`self.lock` 取得 Guard 内部的 `lock` 字段，类型是：

```rust
&RwLock<T>
```

Rust 会自动对引用进行解引用，以便访问字段，因此通常不需要写成：

```rust
(*self).lock
```

这属于 Rust 的自动解引用行为。这里真正需要显式写 `*` 的，是
`data.get()` 返回的裸指针。

## 8. `*` 的其他常见含义

### 乘法

```rust
let area = width * height;
```

当 `*` 出现在两个数值表达式之间时，它表示乘法。

### 解构指针或引用

```rust
let reference = &value;
match reference {
    &number => println!("{}", number),
}
```

在模式中，`&number` 表示匹配一个引用并取出其指向的值。表达式中的
`*reference` 和模式中的 `&number` 都与引用解构有关，但语法位置不同。

### 指针类型声明

```rust
let a: *const i32;
let b: *mut i32;
```

这里的 `*` 不是解引用，而是类型语法的一部分，表示裸指针类型。

## 9. `&` 的其他常见含义

### 位与运算

```rust
let result = 0b1100u8 & 0b1010u8;
assert_eq!(result, 0b1000);
```

当 `&` 出现在两个整数表达式之间时，它表示按位与；只有出现在值前面
时，通常才表示借用。

### 引用类型声明

```rust
let shared: &i32 = &value;
let mutable: &mut i32 = &mut value;
```

左侧的 `&` 是类型语法，右侧的 `&` 是创建引用的操作。

## 10. 常见错误对比

### 直接返回裸指针

```rust
fn get(&self) -> &T {
    self.data.get() // 错误：*mut T 不是 &T
}
```

必须将裸指针转换为引用：

```rust
unsafe { &*self.data.get() }
```

### 忘记 `unsafe`

```rust
&*self.data.get() // 解引用裸指针需要 unsafe
```

正确写法：

```rust
unsafe { &*self.data.get() }
```

### 把 `&mut T` 与 `&T` 混淆

```rust
unsafe { &mut *ptr } // 返回 &mut T
unsafe { &*ptr }     // 返回 &T
```

可变引用具有更强的独占要求，不能在仍有其他共享引用时创建。

## 11. 记忆方法

可以用下面的顺序理解相关表达式：

```text
&value       借用 value，得到 &T
*reference   访问 reference 指向的 T
&*pointer    解引用裸指针，再借用为 &T
&mut *pointer
             解引用裸指针，再借用为 &mut T
```

对于当前读写锁：

```rust
unsafe { &*self.lock.data.get() }
```

最终目标就是把：

```text
*mut T
```

转换成：

```text
&T
```

而这一步之所以需要 `unsafe`，是因为 Rust 无法仅凭裸指针判断地址有效性
以及读写锁协议是否正确，安全责任由实现者承担。
