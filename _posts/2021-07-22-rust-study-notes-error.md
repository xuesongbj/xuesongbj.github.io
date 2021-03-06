---
layout: post
title: 'rust异常处理'
tags: [code]
---

Rust中大多数错误处理都是通过 `Option` 和 `Result`这两种通用类型完成的。 `Option` 和 `Result` 类型是标准库中的普通类型，这意味着它们不是编译器内部类型，所以编译器会将它们和其他函数一视同仁。任何人都可以使用枚举和泛型功能创建类似的错误抽象类型。

大多数异常处理一般有三种。

* 可恢复异常。例如文件未找到(file not found)或数字解析错误。
* 不可恢复异常。例如索引越界或除以非零的操作(分母是零)。
* 致命性异常。包括内存不足和堆栈溢出。

&nbsp;

## Result

通常使用 `Result` 的枚举对象作为程序的返回值，通过`Result`来判断其结果，我们使用`match`匹配的方式来获取`Result`的内容，判断正常`(Ok)`或错误`(Err)`。

Result是一个`enum`枚举对象，具体源码:

```rust
pub enum Result<T, E> {
    /// Contains the success value
    Ok(#[stable(feature = "rust1", since = "1.0.0")] T),

    /// Contains the error value
    Err(#[stable(feature = "rust1", since = "1.0.0")] E),
}
```

&nbsp;

## Error具体实现

Error 是一个`trait`，具体由Display、source方法实现。

```rust
// library/std/src/error.rs

#[stable(feature = "rust1", since = "1.0.0")]
pub trait Error: Debug + Display {
    #[stable(feature = "error_source", since = "1.30.0")]
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
`
    #[stable(feature = "rust1", since = "1.0.0")]
    #[rustc_deprecated(since = "1.42.0", reason = "use the Display impl or to_string()")]
    fn description(&self) -> &str {
        "description() is deprecated; use Display"
    }
}
```

* description: 该方法不再推荐使用，而是由Display代替。
* source：进行错误输出，有错误则返回`Err`，返回Some(e);如果没有返回`None`。

&nbsp;

### 异常处理简单示例

```rust
fn read_file(path: &str) -> Result<String, std::io::Error> {    // Result作为结果返回值
    std::fs::read_to_string(path)                               // 读取文件内容
}

fn main() {
    let path = "/tmp/data";     // 文件路径
    match read_file(path) {
        Ok(file)    =>  { println!("{}", file) }            // OK 代表读取到文件内容，正确打印文件内容
        Err(e)      =>  { println!("{} {}", path, e) }      // Err代表结果不存在，打印错误结果
    }
}
```

&nbsp;

#### 自定义Error

自定义的`Error`需要`impl std::fmt::Debug`的`trait`。具体实现一个自定义Error需要以下几步骤:

* 实现impl `std::fmt::Display`的`trait`, 并实现`fmt(...)`方法。
* 实现impl `std::fmt::Debug`的`trait`, 一般直接添加注释即可: `#[derive(Debug)]`。
* 实现impl `std::error::Error`的`trait`，并根据自身`error`级别是否**覆盖** `std::error::Error`中的`source()`方法。

&nbsp;

##### 自定义错误: SimpleError

```rust
use std::error::Error;

// 1. 自定义类型Error，实现 std::fmt::Debug的trait
#[derive(Debug)]
struct SimpleError {
    err: ChildError,
}

// 2. 实现Display的trait，并实现fmt方法
impl std::fmt::Display for SimpleError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "SimpleError!!!")
    }
}

// 3. 实现Error的trait，因为有子Error::ChildError，需要覆盖source()方法，返回Some(err)
impl std::error::Error for SimpleError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        Some(&self.err)
    }
}
```

##### 子类型错误ChildError:

```rust
// 4. 子类型Error，实现std::fmt::Debug的trait
#[derive(Debug)]
struct ChildError;

// 5. 实现Display的trait，并实现fmt方法
impl std::fmt::Display for ChildError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "ChildError!!!")
    }
}

// 6. 实现Error的trait,因为没有子Error,不需要覆盖source()方法
impl std::error::Error for ChildError {}
```

##### 使用自定义Error

```rust
// 构建一个Result的结果，返回自定义的error:SimpleError
fn get_super_error() -> Result<(), SimpleError> {
    Err(SimpleError { err: ChildError })
}

fn main() {
    match get_super_error() {
        Err(e) => {
            println!("Error: {}", e);
            println!("Caused by: {}", e.source().unwrap());
        }
        _ => println!("No error"),
    }
}
```

* `ChildError`为子类型`Error`，**没有覆盖**`source()`方法。
* `SimpleError`有子类型`ChildError`，**覆盖了** `source()`，并返回了类型Option值`Some(&self.err)`

&nbsp; &nbsp;

### 自定义错误转换: From

可以将多个**单独的Error** 放到一个**自定义Error** 中，将多个**单独的Error** 类型变成**自定义Error** 的子Error，这样对外的Result统一返回**自定义Error**。

```rust
#[derive(Debug)]
enum CustomError {
    ParseIntError(std::num::ParseIntError),
    Utf8Error(std::str::Utf8Error),
    IoError(std::io::Error),
}

impl std::error::Error for CustomError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static')> {
        match &self {
            CustomError::IoError(ref e)     => Some(e),
            CustomeError::Utf8Error(ref e)  => Some(e),
            CustomeError::ParseIntError(ref f) => Some(e),
        }
    }
}

import Display for CustomError {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        match &self {
            CustomError::IoError(ref e)     => e.fmt(f),
            CustomeError::Utf8Error(ref e)  => e.fmt(f),
            CustomeError::ParseIntError(ref f) => e.fmt(f), 
        }
    }
}

impl From<ParseIntError> for CustomError {
    fn from(s: std::num::ParseIntError) -> Self {
        CustomError::ParseIntError(s)
    }
}

impl From<IoError> for CustomError {
    fn from(s: std::io::Error) -> Self {
        CustomError::IoError(s)
    }
}

impl From<Utf8Error> for CustomError {
    fn from(s: std::str::Utf8Error) -> Self {
        CustomError::Utf8Error(s)
    }
}
```

* `CustomError`实现了自定义`Error`
* `CustomError`有三个子类型`Error`
* `CustomError`分别实现了三个**子类型Error**`From`的trait,将其类型包装为**自定义Error的子类型**

#### 使用自定义Error

```rust
use std::io::Error as IoError;
use std::str::Utf8Error;
use std::num::ParseIntError;
use std::fmt::{Display, Formatter};

fn main() -> std::result::Result<(), CustomError>{
    // 1. 读取一个文件
    // 2. 转成utf8
    // 3. 转成u32

    let path = "./dat";
    let v = read_file(path)?;
    let x = to_utf8(v.as_bytes())?;
    let u = to_u32(x)?;
    println!("num:{:?}", u);

    Ok(())
}

// 读取文件内容
fn read_file(path: &str) -> std::result::Result<String, std::io::Error> {
    std::fs::read_to_str(path)
}

// 转换成utf8内容
fn to_utf8(v: &[u8]) -> std::result::Result<&str, std::str::Utf8Error> {
    std::str::from_utf8(v)
}

// 转成为u32数字
fn to_u32(v: &str) -> std::result::Result<u32, std::num::ParseIntError> {
    v.parse::<u32>()
}
```

&nbsp;
&nbsp;

使用`?`代替原来的`match`匹配方式。`?`使用问号作用在函数的结束，意思是:

* 接受了一个`Result<(), CustomError>` 错误类型
* 如果函数返回错误，则抛出`Err`异常，包含CustomError错误信息(因为`impl  Form`转换的操作，该函数的自身类型错误会通过实现的From操作自动转化为CustomError的自定义类型错误),退出当前函数。
* 如果函数返回值正常，继续之后逻辑，直到程序结束

&nbsp;
&nbsp;

### 重命名Result

在项目中，会大量使用`Result`结果，并且`Result`的`Err`类型是我们**自定义错误**，导致我们写程序时会显得**啰嗦、冗余**。

```rust
// 读取文件内容
fn read_file(path: &str) -> std::result::Result<String, CustomError> {
    let val = std::fs::read_to_string(path)?;
    Ok(val)
}

// 转换为utf8内容
fn to_utf8(v: &[u8]) -> std::result::Result<&str, CustomError> {
    let x = std::str::from_utf8(v)?;
    Ok(x)
}

// 转化为u32数字
fn to_u32(v: &str) -> std::result::Result<u32, CustomError> {
    let i = v.parse::<u32>?;
    Ok(i)
}
```

在项目中，大量充斥着这种**模版代码**，Rust本身支持对类型自定义，只需要重命名`Result`即可。

```rust
// 自定义Result类型：IResult
pub type IResult<I> = std::result::Result<I, CustomError>;
```

&nbsp;

这样，凡是使用的是自定义类型错误的Result都可以使用IResult来替换std::result::Result的类型，使得简化程序，隐藏Error类型及细节。

```rust
// 读取文件内容
fn read_file(path: &str) -> IResult<String> {
    let val = std::fs::read_to_string(path)?;
    Ok(val)
}

// 转换为utf8内容
fn to_utf8(v: &[u8]) -> IResult<&str> {
    let x = std::str::from_utf8(v)?;
    Ok(x)
}

// 转化为u32数字
fn to_u32(v: &str) -> IResult<u32> {
    let i = v.parse::<u32>?;
    Ok(i)
}
```

将`std::result::Result<I, CustomError>` 替换为 `IResult<I>`类型。

&nbsp;

如果多参数类型，只需要将`OK`类型变成`tuple(I, O)`类型的多参数数据即可。

```rust
pub type IResult<I, O> = std::result::Result<(I, O), CustomError>;

// 具体使用
fn foo() -> IResult<String, u32> {
    Ok((String::from("bar"), 32))
}
```

