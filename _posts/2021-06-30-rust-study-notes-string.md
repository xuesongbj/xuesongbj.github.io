---
layout: post
title: 'rust数据-字符串'
tags: [code]
---

字符串(UTF-8)相关类型，根据用途选用。

* `char`: 字符，对应Unicode码点。
* `str`: 字符串切片(`[u8]`), 动态尺寸类型(DST)。
* `String`: 堆分配可变字符串(`Vec<u8>`)。

&nbsp;

## 使用建议

* 字面量无法修改，直接对应`&str`。
* 要动态修改的字符串使用`String`。
* 返回子串，**不想获取所有权，不想复制内容**，用`&str`或`&mut str`。

&nbsp;

## 结构示意图

```rust

    String               heap
   +=======+           +=========//========+
   | ptr   | --------> | u8 data ...       |
   +-------+           +=========//========+
   | cap   |
   +-------+
   | len   |
   +=======+
  

    &str
   +=======+           +========//========+
   | ptr   | --------> | u8 data ....     |
   +-------+           +========//========+
   | len   |
   +=======+
   
```

```rust
struct String {
  vec: Vec<u8> {
    buf: RawVec<u8> {
      ptr,
      cap,
    },
      len,
    }
}

struct &str {
  ptr: *mut u8,
  len: usize,
}

struct &mut str {
  ptr: *mut u8,
  len: usize,
}
```

&nbsp;

## 字面量

静态分配(.rodata)。

* 跨行: 支持跨行。若以`\`结尾，则删除换行和前置空格。
* 转义: `abc, \x41 \u{6211}们 \" \\`
* 原始字符串(raw string): `r"a\x41bc"`，`r#"a"bc"#`(内容包含引号)，`r##"a#"bc##`(内容包含#, 用更多的##作边界)

```rust
// 字面量
let s: &'static str = "hello, world!";
```

```rust
(gdb) p/x &s
$4 = 0x7fffffffe3a8

// 变量s数据存储在0x000055555558c000位置，即.rodata section
(gdb) x/xg &s                                                           
0x7fffffffe3a8:	0x000055555558c000

(gdb) info proc  mappings
0x555555559000     0x55555558c000    0x33000     0x5000 /root/rs/ddd/target/debug/ddd

(gdb) x/16xb 0x000055555558c000                                         // 底层具体存储
0x55555558c000:	0x68	0x65	0x6c	0x6c	0x6f	0x2c	0x20	0x77
0x55555558c008:	0x6f	0x72	0x6c	0x64	0x0a	0x00	0x00	0x00
```

&nbsp;

rust编译器会将换行的 `\`、`空格` 符号删除。

```rust
fn main() {
    // 字面量
    let s = "foo\
             bar";

    println!("{:?}", s);
    assert_eq!("foobar", s);
}
```

&nbsp;

### 字节数组

添加`b`前缀，表示`Byte` String，同样支持转移、跨行和原始字符串(rb)。但也意味着，只能是ASCII字符，无法容纳Unicode字符。

```rust
fn main() {
    let s = b"abc";
    assert_eq!(s, &[b'a', b'b', b'c']);
}
```

```rust
(gdb) info locals
s = 0x55555558b000 b"abc"                   // abc存储位置: 0x55555558b000，即.rodata

(gdb) info proc mappings
0x555555559000     0x55555558b000    0x32000     0x5000 /root/rs/ddd/target/debug/ddd

(gdb) x/3xb 0x55555558b000                  // 具体存储内容 -> 'abc'
0x55555558b000:	0x61	0x62	0x63
```

&nbsp;

## 转换

根据需要，在不同类型间转换。

&nbsp;

### 字面量转字符串

```rust
// literal -> string, heap_alloc

fn main() {
    let mut s = 1.to_string();

    let mut s = "abc".to_string();

    let mut s = String::from("abc");

    println!("{:?}", s);
}
```

反汇编:

```rust
// let mut s = 1.to_string();

(gdb) info locals
s = alloc::string::String {
  vec: alloc::vec::Vec<u8, alloc::alloc::Global> {
    buf: alloc::raw_vec::RawVec<u8, alloc::alloc::Global> {
      ptr: core::ptr::unique::Unique<u8> {
        pointer: 0x5555555a59d0 "1\000",
        _marker: core::marker::PhantomData<u8>
      },
      cap: 8,
      alloc: alloc::alloc::Global
    },
    len: 1
  }
}

(gdb) info proc mappings                                                // "1"被分配在heap上
0x5555555a5000     0x5555555c6000    0x21000        0x0 [heap]

(gdb) x/xg 0x5555555a59d0
0x5555555a59d0:	0x0000000000000031                                      // ASCII 31 -> 1


// let mut s = "abc".to_string();
(gdb) info locals
s = alloc::string::String {
  vec: alloc::vec::Vec<u8, alloc::alloc::Global> {
    buf: alloc::raw_vec::RawVec<u8, alloc::alloc::Global> {
      ptr: core::ptr::unique::Unique<u8> {
        pointer: 0x5555555a59f0 "abc\000",                              // "abc"被分配在heap上 
        _marker: core::marker::PhantomData<u8>
      },
      cap: 3,
      alloc: alloc::alloc::Global
    },
    len: 3
  }
}

(gdb) x/3xb 0x5555555a59f0
0x5555555a59f0:	0x61	0x62	0x63

// let mut s = String::from("abc");
(gdb) info locals
s = alloc::string::String {
  vec: alloc::vec::Vec<u8, alloc::alloc::Global> {
    buf: alloc::raw_vec::RawVec<u8, alloc::alloc::Global> {
      ptr: core::ptr::unique::Unique<u8> {
        pointer: 0x5555555a5a10 "abc\000",                              // "abc"被分配在heap上 
        _marker: core::marker::PhantomData<u8>
      },
      cap: 3,
      alloc: alloc::alloc::Global
    },
    len: 3
  }
}

(gdb) x/3xb 0x5555555a5a10
0x5555555a5a10:	0x61	0x62	0x63
```

&nbsp;

### String 转 &str

```rust
// let x: &str = &s;
(gdb) ptype x
type = struct &str {
  data_ptr: *mut u8,
  length: usize,
}

(gdb) p/x &x                                // 变量x内存地址(stack)
$1 = 0x7fffffffe3a8

(gdb) x/xg 0x7fffffffe3a8
0x7fffffffe3a8:	0x000055555559f9d0          // 变量x指针地址(heap)

(gdb) x/xg 0x7fffffffe3a8+0x8               // 字符串在heap长度
0x7fffffffe3b0:	0x0000000000000003

(gdb) x/3xb 0x000055555559f9d0              // 字符串内容
0x55555559f9d0:	0x61	0x62	0x63


// let y: &mut str = &mut s;
(gdb) ptype y
type = struct &mut str {
  data_ptr: *mut u8,
  length: usize,
}

(gdb) info locals
y = &mut str {
  data_ptr: 0x55555559f9d0 "abc\000",       // 变量y指针地址(heap)
  length: 3
}

(gdb) x/3xb 0x55555559f9d0                  // 字符串内容
0x55555559f9d0:	0x61	0x62	0x63
```

&nbsp;

### String 转 Unicode Char

```rust
fn main() {
    let s = "我们".to_string();

    let c = s.chars();
    let s2 = c.as_str();

    assert_eq!(s, s2);
}
```

## 操作

### 格式化字符串

```rust
// format

fn main() {
    let s = format!("{}{}", "a", 1);

    assert_eq!(s, "a1");
}
```

&nbsp;

### 链接字符串

```rust
fn main() {
    let v = ["a", "b", "cd"];
    assert_eq!(v.concat(), "abcd");
    assert_eq!(v.join(","), "a,b,cd");
}
```
