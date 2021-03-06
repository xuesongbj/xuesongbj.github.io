---
layout: post
title: 'rust数据-切片'
tags: [code]
---

![](https://raw.githubusercontent.com/xuesongbj/xuesongbj/main/images/rust_02.jpeg)


切片(slice,`[T]`)引用序列对象片段，没有所有权。动态尺寸类型(DST，dynamically sized type)，无法直接使用，只能通过指针(`&[T]`)进行。


> A type with a size that is known only at run-time is called a dynamically sized type (DST) or, informally, an unsized type. Slices and trait objects are two examples of DSTs.


> 数组：`[T; size]` vs `&[T]`切片
> 
> 因为`[T]`长度在编译期间无法确认，所以只能通过固定长度的“指针”定义。
>
> 所谓`&[T]`切片指针，本质上是一个结构体，包含指针和长度信息。

&nbsp;

```rust
fn main() {
    let v = [1, 2, 3];              // array
    // let s: &[i32] = &v[...];     // slice

    let s = v[..];                  // 不支持直接操作切片,需要通过slice pointer进行操作
       //  ^ doesn't have a size known at compile-time
       // note: all local variables must have a statically known size
}
```

当操作动态尺寸(DST)类型数据时，只能通过指针进行操作，不能直接操作。

&nbsp;

#### 数据结构示意图

```x86asm
// 反汇编

s = &[i32] {
  data_ptr: 0x7fffffffe394,
  length: 3
}

(gdb) x/3xw 0x7fffffffe394
0x7fffffffe394:	0x00000001	0x00000002	0x00000003
```

```
   &[T]
  +=====+             +=======//=======+
  | ptr | ----------> | data ...       |
  +-----+             +=======//=======+
  | len |
  +=====+
```

```rust
struct &[T] {
    data_ptr: *mut T,
    length: usize,
}
```

&nbsp;

#### 切片类型

* `&T`: shared slice。
* `&mut [T]`: mutable slice， 通过指针修改slice元素
* `Box<[T]>`: boxed slice， 装箱堆上分配

&nbsp;
&nbsp;

### 构造

从已有序列对象，以**起始**和**结束**索引进行构造。

* `[..]`: `[0, len -1]`
* `[0..2]`: `[0, 2]`
* `[0..=2]`: `[0, 2]`
* `[3..]`: `[3, len-1]`
* `[..3]`: `[0, 3]`
* `[..=3]`: `[0, 3]`

> `&v[..]`的实际操作是`&(v[..])`。
>
> `[T]`是切片；`&[T]`是通过"指针"访问切片。

&nbsp;

```rust
fn main() {
    let v = vec![0, 1, 2, 3];

    let s = &v[..];                     // [0, 3]
    assert_eq!([0, 1, 2, 3], s);

    let s = &v[0..2];                   // [0, 2)
    assert_eq!([0, 1], s);

    let s = &v[0..=2];                  // [0, 2]
    assert_eq!([0, 1, 2], s);

    let s = &v[..2];                    // [0, 2)
    assert_eq!([0, 1], s);

    lset s = &v[..=2];                  // [0, 2]
    assert_eq!([0, 1, 2], s);

    let s = &v[2..];
    assert_eq!([2, 3], s);              // [2, 3]
}
```

&nbsp;


### 类型转换

注意区别`数组引用`和`切片`，虽然可以强制转换。

```rust
fn type_of<T>(_: &T) -> &'static str {
    std::any::type_name::<T>()
}

fn main() {
    let s = &[1, 2, 3];
    assert_eq!("&[i32; 3]", type_of(&s));       // array reference

    let s = &[1, 2, 3][..];
    assert_eq!("&[i32]", type_of(&s));          // slice

    let s: &[i32] = &[1, 2, 3];                 // convert
    assert_eq!("&[i32]", type_of(&s));          // slice

    let s = &["one", "two"][..];                // slice
    assert_eq!("&[&str]", type_of(&s));
}
```

```rust
use std::convert::TryFrom;

fn type_of<T>(_: &T) -> &'static str {
    std::any::type_name::<T>()
}

fn main() {
    let a = &[1, 2, 3];
    let s = &a[..];

    assert_eq!("&[i32; 3]", type_of(&a));       // array
    assert_eq!("&[i32]", type_of(&s));          // slice

    // === convert =====================================

    let s2: &[i32] = &[1, 2, 3];                // array2slice
    assert_eq!("&[i32]", type_of(&s));

    let a2 = <&[i32; 3]>::try_from(s2);         // slice2array
    assert_eq!("&[i32; 3]", type_of(&a2.unwrap()));

    println!("{:?}", &a2);
}
```

&nbsp;

### 操作

通过可变切片间接修改数据源。

&nbsp;

> ⚠️
> 切片索引与数据源索引未必一致。

```rust
fn main() {
    let mut v = vec![0, 1, 2, 3];
    
    let s = &mut v[2..];
    s[0] += 100;                        // 使用slice的切片索引修改原vec数据

    assert_eq!(v, [0, 1, 102, 3]);
}
```

反汇编:

```x86asm
s = &mut [i32] {                                            // slice
  data_ptr: 0x55555559f9d8,
  length: 2
}

v = alloc::vec::Vec<i32, alloc::alloc::Global> {            // vec
  buf: alloc::raw_vec::RawVec<i32, alloc::alloc::Global> {
    ptr: core::ptr::unique::Unique<i32> {
      pointer: 0x55555559f9d0,
      _marker: core::marker::PhantomData<i32>
    },
    cap: 4,
    alloc: alloc::alloc::Global
  },
  len: 4
}
```

&nbsp;

#### 遍历(修改)切片

```rust
fn main() {
    let s = &mut[1, 2, 3][..];

    for x in s.iter_mut() {
        *x += 100;
    }

    for x in s.iter() {
        println!("{:?}", x);
    }

    println!("{:?}", s);
}
```

&nbsp;

源码实现:

```rust
// core/src/slice/mod.rs:723
// 循环遍历获取slice元素

#[inline]
pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    IterMut::new(self)
}

impl<'a, T> IterMut<'a, T> {
    pub(super) fn new(slice: &'a mut [T]) -> Self {
        let ptr = slice.as_mut_ptr();

        unsafe {
            // 检查是否ZST类型
            let end = if mem::size_of::<T>() == 0 {
                // ptr指针不变
                (ptr as *mut u8).wrapping_add(slice.len()) as *mut T
            } else {
                // ptr指针后移
                ptr.add(slice.len())
            };

            // end: 检查迭代器是否完成
            // 依次移动ptr指针位置，直到遍历完成
            Self { ptr: NonNull::new_unchecked(ptr), end, _marker: PhantomData }
        }
    }

    // ...
}
```

```rust
pub fn iter(&self) -> Iter<'_, T> {
    Iter::new(self)
}

impl<'a, T> Iter<'a, T> {
    pub(super) fn new(slice: &'a [T]) -> Self {
        let ptr = slice.as_ptr();
        unsafe {
            let end = if mem::size_of::<T>() == 0 {
                (ptr as *const u8).wrapping_add(slice.len()) as *const T
            } else {
                ptr.add(slice.len())
            };

            Self { ptr: NonNull::new_unchecked(ptr as *mut T), end, _marker: PhantomData }
        }
    }
}
```

> DST: Dynamic Sized Type缩写，动态大小类型。该类型在编译阶段无法确定大小的类型。
> 
> ZST: Zero Sized Type缩写，0大小类型。

&nbsp;

#### 切片relice操作

```rust
fn main() {
    let mut a = [1, 2, 3];

    let s = &mut a[..];
    let s2 = &mut s[1..];

    s2[1] += 100;

    println!("{:?}", s);
    println!("{:?}", a);
}
```

&nbsp;

重新切片受目标`slice`范围限制，而非底层数组。

```rust
fn main() {
    let mut a = [1, 2, 3, 4, 5, 6];

    let s  = &mut a[..3];
    let s2 = &mut s[1..4];
}
```

```bash
$>  cargo r
   Compiling ddd v0.1.0 (/root/rs/ddd)
    Finished dev [unoptimized + debuginfo] target(s) in 2.93s
     Running `/root/rs/ddd/target/debug/ddd`
thread 'main' panicked at 'range end index 4 out of range for slice of length 3', src/main.rs:7:19
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```