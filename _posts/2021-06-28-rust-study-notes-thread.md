---
layout: post
title: 'rust并发-线程'
tags: [code]
---

并发体系由标准库，而非语言提供支持。这意味着，可能有很多不同的产品可供选择。标准库以1:1系统线程来提供并发支持，使用方式和其他语言类似。

```rust
use std::thread;

fn main() {
    let handler = thread::spawn(|| {    // 生成一个新的线程, 返回一个 JoinHandle
        for i in 1..10 {
            println!("{}", i);
        }
    });

    handler.join().unwrap();            // join等待子线程执行完成。
                                        // unwrap() 输出具体值
}
```

&nbsp;

## JoinHandle

spawn函数会返回一个JoinHandle类型的值，我们将其存放在变量handler中。这种类型是自线程的句柄，可用于连接线程 -- 换句话说就是等待它的终止。如果我们忽略线程的JoinHandle类型,就没有办法等待线程。继续解析我们的代码，从main函数退出之前的子线程上调用join方法。

### join

调用Join会阻塞当前线程，并在执行join调用之后的任何代码行之前等待子线程完成。它返回一个Result值。由于我们知道这个线程没有发生`panic`，可以调用`expect`方法获取`Result`中的字符串。如果一个线程正在连接自身或者遇到死锁，那么连接线程可能会失败。在这种情况下，它会返回一个`Err`变量，其值会传递给处理错误的`panic!`宏，不过这种情况下返回的值是Any类型，必须将其转换为适当的类型。

&nbsp;

## 自定义类型

我们还可以通过设置其属性(名称或堆栈大小)来配置线程的API。

```rust
use std::thread;

fn main() {
    // Builder: 线程工程，可用于配置新线程属性.
    // new(): 生成线程基本配置
    let builder = thread::Builder::new().name("name".into()).stack_size(20 * 1024);

    let handler = builder.spawn(|| {
        for i in 1..10 {
            println!("{}", i);
        }
    }).unwrap();

    handler.join().unwrap();
}
```

&nbsp;

### Builder源码实现

Builder 线程工厂源码实现：

```rust
pub struct Builder {
    // 线程名称，目前仅用于出现panic时，区分线程名称.
    name: Option<String>,

    // 线程Stack大小 
    stack_size: Option<usize>,
}
```

&nbsp;

### 线程名称

使用`name()`函数设置线程名称，目前该线程名称仅用于`panic`时，用于识别线程名称。

```rust
use std::thread;

fn main() {
    let child = thread::Builder::new().name("David".into()).stack_size(20 * 1024);

    let handler = child.spawn(|| {
        panic!("Err");
    }).unwrap();

    handler.join().unwrap();
}
```

```bash
thread 'David' panicked at 'Erro', src/main.rs:11:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Any', src/main.rs:14:20
root@8d75790f92f5:~/rs/ddd/src# cat main.rs
```

&nbsp;

## 闭包函数返回值

接收闭包函数返回值。

```rust
use std::thread;

fn main() {
    let handler = thread::spawn(|| {
        100
    });

    assert_eq!(handler.join().unwrap(), 100);
}
```

&nbsp;

鉴于不同线程生命周期的差异，强制转移环境变量所有权。

```rust
fn main() {
    let data = vec![1, 2, 3, 4];

    let handler = thread::spawn(move || {
        data.iter().for_each(|d| println!("{:?}", d));
    });

    handler.join().unwrap();
}
```
