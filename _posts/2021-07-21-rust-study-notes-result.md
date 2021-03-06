---
layout: post
title: 'rust异常处理-可恢复错误'
tags: [code]
---

可恢复性（recoverable）错误通常指一种状态，比如打开文件时，没找到，应该新建。显然，最终如何处理，由调用方负责。没有异常，对于可恢复性错误，返回`Result<T, E>`对象。

&nbsp;

## Option

Rust中没有 `null` 值的概念，在Rust中，可能失败并希望指示缺失相应值的API会返回 `Option`。当任何API及其后续值想要表示缺少值时，此错误类型就是合适的。简而言之，它与`null`值类似，不过这里的 `null` 值检查是显式的，并且在编译期由类型系统强制执行。

Option包含以下标签签名:

```rust
pub enum Option<T> {
    /// 没有值
    None,

    /// 包含某些值 'T'
    Some(T),
}
```

&nbsp;

它是包含两个变体的枚举，并且 `T` 是泛型。操作成功时，可以使用Some(T)存储任意类型值`T`，或者使用 `None` 变量表示操作失败的情况下该值为null。

```rust
let wrapped_i32 = Some(2);          // Some(T)

let empty: Option<i32> = None;      // None

let empty = None::<i32>;            // None
```

&nbsp;

### prelude模块

Rust的标准库，有一个 `prelude` 子模块，该模块自动导入了常用的类型、函数和任意标准库模块的重新导出。导入后，允许直接通过变体实例话 `Option` 值。

目前版本导入以下内容:

```rust
//! [mem::drop]: crate::mem::drop
//! [std::borrow]: crate::borrow
//! [std::boxed]: crate::boxed
//! [std::clone]: crate::clone
//! [std::cmp]: crate::cmp
//! [std::convert]: crate::convert
//! [std::default]: crate::default
//! [std::iter]: crate::iter
//! [std::marker]: crate::marker
//! [std::mem]: crate::mem
//! [std::ops]: crate::ops
//! [std::option]: crate::option
//! [`std::prelude::v1`]: v1
//! [std::result]: crate::result
//! [std::slice]: crate::slice
//! [std::string]: crate::string
//! [std::vec]: mod@crate::vec
//! [`to_owned`]: crate::borrow::ToOwned::to_owned
//! [book-closures]: ../../book/ch13-01-closures.html
//! [book-dtor]: ../../book/ch15-03-drop.html
//! [book-enums]: ../../book/ch06-01-defining-an-enum.html
//! [book-iter]: ../../book/ch13-02-iterators.html
```

&nbsp;

### Option 交互

如下实例，`value` 是一个 `Option<&i32>`。`get()` 方法返回的是一个 `Option<&i32>`，而不是其内部的值(`&i32`)。为了给 `value`的值加1，我们需要从 `Option`中提取 `i32`。为了检查变体，我们有两种方法，他们分别是 `模式匹配` 和 `if let` 语句。

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert("one", 1);
    map.insert("two", 2);

    let incremented_value = match map.get("one") {
        Some(val) => val + 1,
        None => 0
    };

    println!("{}", incremented_value);
}
```

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert("one", 1);
    map.insert("two", 2);

    let incremented_value = if let Some(v) = map.get("one") {
        v + 1
    } else {
        0
    };

    println!("{}", incremented_value);
}
```

当我们对值的某个变体感兴趣并希望对其他变体进行常规的操作时，推荐使用这种方法。`if let` 语句表述更简洁。

&nbsp;

### Unwrapping

`Result<T, E>` 类型定义了很多辅助方法来处理各种情况，其中之一叫做`unwrap`。

* 如果`Result`值是成员`OK`, `unwrap`会返回`OK`中的值。
* 如果`Result`是成员`Err`，`unwrap`会为我们调用`panic!`。

Unwrapping是一种不太安全的方法在 `Option` 上调用解压缩方法。如果返回的结果是 `Some`，那么调用 `unwrap()` 和 `expect()`方法提取内部的值；如果返回的结果是 `None`，则会发生异常。

**仅当我们确定 `Option` 值确实包含某个值时，才推荐使用这些方法。**

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert("one", 1);
    map.insert("two", 2);

    let incremented_value = map.get("two").unwrap() + 1;
    println!("{}", incremented_value);
}
```

`get`一个不存在的**键值**，则发生异常。

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert("one", 1);
    map.insert("two", 2);

    let incremented_value = map.get("three").unwrap() + 1;
    println!("{}", incremented_value);
}
```

```rust
root@8d75790f92f5:~/rs/ddd/src# cargo r
   Compiling ddd v0.1.0 (/root/rs/ddd)
    Finished dev [unoptimized + debuginfo] target(s) in 5.21s
     Running `/root/rs/ddd/target/debug/ddd`
thread 'main' panicked at 'called `Option::unwrap()` on a `None` value', src/main.rs:12:46
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

&nbsp;

#### unwrap() 和 expect() 区别

* `expect()` 是首选方法，因为它允许你传递一个字符串作为发生异常时输出的提示信息，同时显示源文件发生异常时的确切代码行号。

* `unwrap()` 不允许你将调试信息作为参数进行传递，并显示在标准库源文件中 `Option` 下定义 `unwrap()`的代码行号。

&nbsp;

## Result<T, E>

`Result` 和 `Option` 类似，但具有一些优点，即能够存储和错误上下文有关的异常值，而不只是 `None`。**当我们希望知道操作失败的原因时，此类型是合适的。**

`Result`类型签名:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

&nbsp;

它包含两个变体，并且都是泛型。`Ok(T)`是用于表示成功状态时放入任意值 `T` 的变体，而 `Err(E)` 是用于表示执行失败时放入任何异常值 `T`的变体。

```rust
fn main() {
    // Ok(T)
    let _my_result: Result<_, ()> = Ok(64);
    let _my_result = Ok::<_, ()>(64);

    // Err(E)
    let _my_err = Err::<(), f32>(354.3);
    let _other_err: Result<bool, String> = Err("Wait, What ?".to_string());
}
```

使用 `()` 指定 `Err` 变体的类型 `E`。可以使用 `_`来要求Rust为我们推断具体的类型。

### Result实例

标准库中的许多文件操作API都会返回 `Result` 类型，因为可能存在不同的操作失败原因，例如找不到文件(file not found)、目录不存在(directory does not exists)，以及权限错误(permission errors)等。

```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn main() {
    let path = Path::new("data.txt");
    let mut file = match File::open(&path) {
        Ok(file)    => file,
        Err(err)    => panic!("Error while opening file: {}", err),
    };

    let mut s = String::new();
    let _ = file.read_to_string(&mut s);
    println!("Message: {}", s);
}
```

&nbsp;

### 传播

可通过 `return Err`将错误按调用堆栈传递出去。

```rust
fn div(x: i32, y: i32) -> Result<i32, &'static str> {
    if y != 0 { 
        Ok(x / y) 
    } else { 
        Err("divide by zero") 
    }
}

fn test(x: i32, y: i32) -> Result<i32, &'static str> {
    let z = div(x, y);
    if z.is_err() { return z; }  // 传播！
    z
}

fn main() {
    test(3, 0).unwrap();        // 处理错误！
}                                      
```

```rust
root@8d75790f92f5:~/rs/ddd/src# cargo r
   Compiling ddd v0.1.0 (/root/rs/ddd)
    Finished dev [unoptimized + debuginfo] target(s) in 3.55s
     Running `/root/rs/ddd/target/debug/ddd`
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: "divide by zero"', src/main.rs:22:16
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

&nbsp;


## Option/Result的组合

由于 `Option` 和 `Result` 是包装器类型，因此安全地与其内部值进行交互的唯一方法是通过**模式匹配**或**if let语句**。这些包装器类型附带了很多辅助方法，这些被称为**组合器**，其上的实现允许用户轻松地操作内部值。

这些是泛型方法，根据使用场景不同而包含多种，例如 `Ok(T)/Some(T)`，而其中一些方法用于处理失败的值，例如 `Err(E)/None`。某些方法用于拆解和提取内部值，而有些方法会保留包装器类型的结构，仅修改内部值。

> 注意:
>
> 成功值时，通常指的是 `Ok(T)/Some(T)`变量；失败值时，通常指的是 `Err(T)/None(T)` 变量。

&nbsp;

### 常见的组合器

`map()` 此方法允许你将表示成功的值 `T` 转换为另一个值 `U`。以下是 `Option` 类型对应 `map` 特征签名:

```rust
pub fn map<U, F>(self, f: F) -> Option<U>
where
    F: FnOnce(T) -> U
{
    match self {
        Some(x) => Some(f(x)),
        None => None,
    }
}
```

&nbsp;

以下是和Result类型有关的签名:

```rust
pub fn map<U, F>(self, f: F) -> Option<U>
where F: FnOne(T) -> U {
    match self {
        Ok(t) => Ok(f(t)),
        Err(e) => Err(e)
    }
}
```

这是特征的某种特殊类型，仅适用于闭包，因此具有 `(T) -> U` 这样的函数签名。`FnOnce` 前缀表示此闭包取得输入参数 `T` 的所有权，并且我们只能用 `T` 调用此闭包一次，因为 `T` 将在执行调用后被使用。

&nbsp;

### 组合器应用

```rust
fn get_nth(items: &Vec<usize>, nth: usize) -> Option<usize> {
    if nth < items.len() {
        Some(items[nth])
    } else {
        None
    }
}

fn double(val: usize) -> usize {
    val * val
}

fn main() {
    let items = vec![7, 6, 5, 4, 3, 5, 3, 10, 3, 2, 4];
    println!("{}", items.len());

    let doubled = get_nth(&items, 4).map(double);
    println!("{:?}", doubled);
}
```

&nbsp;

还可以提供一个内联形式的闭包，如下所示:

```rust
let doubled = get_nth(&items, 4).map(|v| v*v);
```

这是链式调用的简易形式。并且比之前使用的 `match` 表达式和 `if let`语句更简单。

&nbsp;

## Option 和 Result 类型之间转换

还有一些方法可以将一种包装器转换成另一种包装器类型。

* `ok_or`: 将 `Option`值转换为`Result`值，同时将错误值作为第二个参数进行接收。此类型的变体是`ok_or_else`方法，但优于此方法，因为它接收闭包来进行惰性求值。

* `ok`: 此方法将`Result`转化为调用`self`的Option，并且会丢弃`Err`值。

```rust
fn main() {
    let x = Some("foo");
    let _y = x.ok_or(0);
    assert_eq!(x.ok_or(0), Ok("foo"));

    println!("xxx");
}
```

```rust
_y = core::result::Result<&str, i32>::Ok("foo")
x = core::option::Option<&str>::Some("foo")
```

&nbsp;

## 运算符 "?"

使用`?`来代替原来的`match`匹配的方式。`?`作用在函数的结束。

它的运行机制如下: 当我们获得一个成功值时，希望立即提取它；当我们获得一个错误值时，希望提前返回，并将错误传播给调用方。

* 要么解构出`Ok`关联值，要么`return Err`向外传播。
* 只能用于`Result<T, E>`返回值的函数。
* 隐式传播的`Err`关键值类型必须相同，或可自动转换。

```rust
fn div(x: i32, y: i32) -> Result<i32, &'static str> {
    if y != 0 {
        Ok(x/ y)
    } else {
        Err("divide by zero")
    }
}

fn test(x: i32, y: i32) -> Result<i32, &'static str> {
    let z: i32 = div(x, y)?;            // z返回值为i32或者Err
                                        // 如果div函数结果正确，继续之后逻辑，直到程序结束

    Ok(z)                               // Ok将成功值，解构为Result进行返回
}

fn main() {
    test(3, 0).unwrap();
}
```

### 具体实例

它常用的 `match` 表达式来处理 `Result`类型:

```rust
use std::string::FromUtf8Error;

fn str_upper_match(str: Vec<u8>) -> Result<String, FromUtf8Error> {
    let ret = match String::from_utf8(str) {
        Ok(str) => str.to_uppercase(),
        Err(err) => return Err(err)
    };

    println!("Conversion succeeded: {}", ret);
    Ok(ret)
}

fn main() {
    let invalid_str = str_upper_match(vec![197, 198]);
    println!("{:?}", invalid_str);
}
```

运算符 `?` 抽象了这种模式，这使我们能够以一种更简洁的方式编写 `bytes_to_str` 方法。

```rust
use std::string::FromUtf8Error;

fn str_upper_concise(str: Vec<u8>) -> Result<String, FromUtf8Error> {
    let ret = String::from_utf8(str).map(|s| s.to_uppercase())?;
    println!("Conversion succeeded: {}", ret);
    Ok(ret)
}

fn main() {
    let valid_str = str_upper_concise(vec![121, 97, 89]);
    println!("{:?}", valid_str);
}
```

&nbsp;

### 链式调用

```rust
File.open("hello.txt")?.read_to_string(&mut s)?;
```
