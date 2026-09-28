# 进程创建与管道通信

本练习位于 `exercises/01_concurrency_sync/04_process_pipe/src/lib.rs`，使用 Rust 的 `std::process` 和 `std::io` 接口创建子进程，并通过管道与子进程交换数据。

## 1. 使用 `Command` 创建进程

`Command` 用于配置要执行的程序及其参数：

```rust
let output = Command::new("echo")
    .args(["hello"])
    .output()
    .unwrap();
```

调用 `.output()` 会启动子进程，等待它结束，并返回 `Output`：

- `stdout`：标准输出字节。
- `stderr`：标准错误字节。
- `status`：进程退出状态。

标准输出是 `Vec<u8>`，如果确定内容是 UTF-8，可以使用 `String::from_utf8`；不确定时可以使用 `String::from_utf8_lossy`。

## 2. 配置标准输入输出管道

```rust
let mut child = Command::new("cat")
    .stdin(Stdio::piped())
    .stdout(Stdio::piped())
    .spawn()
    .unwrap();
```

`Stdio::piped()` 会为子进程的标准输入或标准输出创建管道。`spawn()` 立即启动子进程，并返回 `Child`。

通过 `take()` 取得管道句柄：

```rust
let mut stdin = child.stdin.take().unwrap();
let mut stdout = child.stdout.take().unwrap();
```

`take()` 会把句柄从 `Child` 中取出，避免同时存在多个所有者。

## 3. 向子进程发送数据

```rust
stdin.write_all(input.as_bytes()).unwrap();
drop(stdin);
```

`write_all` 将输入数据写入子进程的 stdin。写完后必须关闭 stdin，通常通过 `drop(stdin)` 完成。

关闭写端会向子进程发送 EOF。对于 `cat` 或 `grep` 这类需要读到输入结束才退出的程序，如果不关闭 stdin，子进程会一直等待更多数据，父进程读取 stdout 也可能一直阻塞。

## 4. 读取子进程输出

```rust
let mut output = String::new();
stdout.read_to_string(&mut output).unwrap();
```

`read_to_string` 需要传入可变的 `String` 缓冲区，并返回读取的字节数。读取会持续到 stdout 管道关闭，也就是子进程结束并关闭输出端。

读取结束后可以等待子进程：

```rust
child.wait().unwrap();
```

`wait()` 回收子进程并取得退出状态，避免留下僵尸进程。

## 5. 获取退出码

```rust
let status = Command::new("sh")
    .args(["-c", command])
    .status()
    .unwrap();

status.code().unwrap_or(0)
```

`.status()` 只等待进程结束，不捕获标准输出。`ExitStatus::code()` 返回正常退出时的整数退出码；如果进程被信号终止，则可能返回 `None`。

常见约定是：

- `0` 表示成功。
- 非 `0` 表示失败。

## 6. 正确传播错误

对于返回 `io::Result<String>` 的函数，不应使用 `unwrap()` 吞掉错误：

```rust
let output = Command::new(program)
    .args(args)
    .stdout(Stdio::piped())
    .output()?;

String::from_utf8(output.stdout)
    .map_err(|error| io::Error::new(io::ErrorKind::InvalidData, error))
```

这里的 `?` 会把创建或执行进程时的 `io::Error` 返回给调用者。`String::from_utf8` 失败时，则将 UTF-8 错误转换为 `io::Error`。

## 7. 使用 `grep` 过滤管道输入

创建带参数的过滤进程：

```rust
let mut child = Command::new("grep")
    .arg(pattern)
    .stdin(Stdio::piped())
    .stdout(Stdio::piped())
    .spawn()
    .unwrap();
```

父进程将多行文本写入 stdin，关闭 stdin 后，`grep` 读取到 EOF 并输出匹配行。父进程只需读取 stdout，即可获得已经过滤的结果。

这种方式把过滤工作交给子进程完成，而不是在父进程中重复实现匹配逻辑。

## 8. 核心要点

1. `Command` 用于配置和启动子进程。
2. `Stdio::piped()` 将子进程的 stdin/stdout 暴露为可读写管道。
3. 写完 stdin 后必须关闭它，以发送 EOF。
4. 读取 stdout 直到结束后，再用 `wait()` 回收子进程。
5. `status()` 用于获取退出状态，`output()` 同时捕获输出。
6. 返回 `Result` 的函数应使用 `?` 传播错误，而不是直接 `unwrap()`。
7. 管道通信依赖文件描述符和所有权管理，正确关闭句柄是避免阻塞的关键。
