---
layout: post
title: 'rust指针'
tags: [code]
---

很多时候，指针是相对宽泛概念，并非特指**原始指针(raw-pointer)**。以 **引用(reference)** 为例，在经过编译器检查后，其身上所承担的职责已经卸去。起码在底层指令层面，和原始指针并无区别。所以说它是指针确实算不得错。

**引用和原始指针是独立对象**，间接(*p)操作目标。透过指针修改目标值时，无需声明自身为可变(mut)，除非打算改变自己。

![指针](https://github.com/xuesongbj/xuesongbj.github.io/blob/master/_posts/ptr.png?raw=true)

&nbsp;

```rust
fn main() {
    // 要改变的目标，必须是mut。
    let mut x = 100;

    // r: &mut i32，表明目标类型可变。
    let r = &mut x;

    *r+=1;

    // r: &i32, 不可变引用
    let r = &x;
    assert_eq!(*r, 101);
}
```

&nbsp;

## 原始指针

原始指针分`*mut T`和`*const T`两种，分别对应`可变`和`不可变`。**无论读写都不安全，须自行负责。**

* `*const T` : 表示指向类型`T`的不可变原始指针。它是`Copy`类型。类似于`&T`，只是它可以为空值。

* `*mut T` : 一个指向类型`T`的可变原始指针，它不支持`Copy`特性(non-Copy)。 

```rust
fn main() {
    let mut x = 100;

    let r: *mut i32 = &mut x;

    unsafe {
        *r += 1;
        assert_eq!(*r, 101);
    }

    let r: *const i32 = &x;
    assert_eq!(unsafe{ *r }, 101);
}
```

&nbsp;

### 引用和指针互相转换

可以和引用相互转换。

```rust
fn main() {
    let mut x = 100;
    let r = &mut x;

    // 引用转指针必然是安全的。
    let p = r as *mut i32;
    assert_eq!(unsafe{ *p }, x);

    // 指针转回引用就未必安全了。
    let r2: &mut i32 = unsafe{ &mut *p };  // unsafe{ & *p } -> &i32
    *r2 += 1;
    assert_eq!(x, 101);
}
```

```x86asm
(gdb) info locals
r2 = 0x7fffffffe2a4
p = 0x7fffffffe2a4
r = 0x7fffffffe2a4
x = 101

// 从底层实现来看，引用和指针没有区别
(gdb) x/xg 0x7fffffffe2a4
0x7fffffffe2a4:	0x0000000000000064

(gdb) x/xg 0x7fffffffe2a4
0x7fffffffe2a4:	0x0000000000000064

// *r2 += 1;
(gdb) x/wx 0x7fffffffe2a4
0x7fffffffe2a4:	0x00000065
```

&nbsp;

### usize指针转换

可通过`usize`转换，实现指针运算。也正因为如此，指向无法保证。

```rust
use std::mem::size_of_val;

fn main() {
    let mut x = [1, 2, 3, 4];
    let mut p: *mut i32 = &mut x[0]; // 指针自身和目标都可变
    
    unsafe {
        assert_eq!(*p, 1);           // 断言x[0]是否等于1

        p = ((p as usize) + size_of_val(&x[0])) as *mut i32; // p++
        *p += 10;
        assert_eq!(*p, 12);
    }
}
```

&nbsp;

原始指针不参与对象生命周期，也就是说可能成为`悬垂指针`。

```rust
#[derive(Debug)]
struct Data {
    x: i32,
}

impl Drop for Data {
    fn drop(&mut self) {
        println!("Dropping!");
    }
}

fn main() {
    let d = Data{ x: 10 };
    let p = &d as *const Data;

    {
        let _d2 = d;  // move!!!
    }                 // drop!!!

    unsafe {
        println!("{:p}, {:?}", p, *p);
    }
}
```

```rust
$  cargo r
   Compiling ddd v0.1.0 (/root/rs/ddd)
    Finished dev [unoptimized + debuginfo] target(s) in 3.04s
     Running `/root/rs/ddd/target/debug/ddd`
Dropping!
0x7ffdd34821e4, Data { x: 10 }      // 虽然能返回内容，但这只是因为是栈内存未被覆盖的缘故
```

&nbsp;

通过以下反汇编结果，可以看出原始变量`d`在内层超出`{ ... }`作用域后，进行了所有权转移，调用 `drop`进行释放。`main`函数结束时，超出该作用域没有调用`drop`函数释放局部变量`p`，则p成为了**悬垂指针**。

```x86asm
; /root/rs/ddd/src/main.rs:
  fn main() {
  0x0000555555559250 <+0>: sub    rsp,0xb8

  let d = Data{ x: 10 };
  0x0000555555559257 <+7>: mov    DWORD PTR [rsp+0x34],0xa

  let p = &d as *const Data;
  0x000055555555925f <+15>:  lea    rax,[rsp+0x34]
  0x0000555555559264 <+20>:  mov    QWORD PTR [rsp+0x38],rax

  {
    let _d2 = d;  // move!!!
  0x0000555555559269 <+25>:  mov    ecx,DWORD PTR [rsp+0x34]
  0x000055555555926d <+29>:  mov    DWORD PTR [rsp+0x44],ecx
  }              // drop!!!

  0x0000555555559271 <+33>:  lea    rdi,[rsp+0x44]
  0x0000555555559276 <+38>:  call   0x5555555591f0 <core::ptr::drop_in_place>

  ; ...
  
  0x0000555555559346 <+246>:  lea    rdi,[rsp+0x48]
  0x000055555555934b <+251>:  call   QWORD PTR [rip+0x44a1f]        # 0x55555559dd70

    }
  }
  
  0x0000555555559351 <+257>:  add    rsp,0xb8
  0x0000555555559358 <+264>:  ret
; End of assembler dump.
```

&nbsp;
&nbsp;

## 智能指针

在C++中广泛采用智能指针，主要是由于原始指针非常不安全，开发者在使用它们时需要注意很多细节。不恰当地使用它们，可能会以非常隐蔽的方式导致诸如内存泄漏、引用挂起，以及大型代码库中的双重释放等问题。

智能指针(smart-pointer)是一类数据结构，其行为类似指针，拥有额外的功能。与引用的差别在于，引用 **借用(borrow)** 目标，而智能指针则 **拥有(own)** 目标。

智能指针通常实现`Deref`和`Drop`特征。`Deref`重载解引用运算符，让它的操作和引用一致；`Drop`则负责在离开作用域时清理资源。

### Drop

`Drop`可以自动释放相关值超出作用域后占用的资源。它包含一个 `drop` 方法，当对象超出作用域时，就会被调用。该方法将 `&mut self` 作为参数。使用 `drop` 释放值是以LIFO的方式进行的。也就是说，无论最后构建的是什么，都首先会被释放。

```rust
struct Character {
    name: String,
}

impl Drop for Character {
    fn drop(&mut self) {
        println!("{} went away", self.name);
    }
}

fn main() {
    let _steve = Character {
        name: "Steve".into(),
    };

    let _john = Character {
        name: "John".into(),
    };
}
```

```rust
root@8d75790f92f5:~/rs/ddd/src# cargo r
   Compiling ddd v0.1.0 (/root/rs/ddd)
    Finished dev [unoptimized + debuginfo] target(s) in 3.93s
     Running `/root/rs/ddd/target/debug/ddd`

John went away
Steve went away
```

```x86asm
(gdb) disassemble
Dump of assembler code for function ddd::main:
   0x0000555555559a10 <+0>:	  sub    rsp,0x78
   0x0000555555559a14 <+4>:	  lea    rax,[rip+0x34640]        # 0x55555558e05b
   0x0000555555559a1b <+11>:	lea    rdi,[rsp+0x18]
   0x0000555555559a20 <+16>:	mov    rsi,rax
   0x0000555555559a23 <+19>:	mov    edx,0x5
   0x0000555555559a28 <+24>:	call   0x55555555a610 <<T as core::convert::Into<U>>::into>
   0x0000555555559a2d <+29>:	mov    rax,QWORD PTR [rsp+0x28]
   0x0000555555559a32 <+34>:	mov    QWORD PTR [rsp+0x10],rax
   0x0000555555559a37 <+39>:	movups xmm0,XMMWORD PTR [rsp+0x18]
   0x0000555555559a3c <+44>:	movaps XMMWORD PTR [rsp],xmm0
   0x0000555555559a40 <+48>:	lea    rsi,[rip+0x34619]        # 0x55555558e060
   0x0000555555559a47 <+55>:	lea    rdi,[rsp+0x50]
   0x0000555555559a4c <+60>:	mov    edx,0x4
   0x0000555555559a51 <+65>:	call   0x55555555a610 <<T as core::convert::Into<U>>::into>
   0x0000555555559a56 <+70>:	jmp    0x555555559a58 <ddd::main+72>
   0x0000555555559a58 <+72>:	mov    rax,QWORD PTR [rsp+0x60]
   0x0000555555559a5d <+77>:	mov    QWORD PTR [rsp+0x40],rax
   0x0000555555559a62 <+82>:	movups xmm0,XMMWORD PTR [rsp+0x50]
   0x0000555555559a67 <+87>:	movaps XMMWORD PTR [rsp+0x30],xmm0
   0x0000555555559a6c <+92>:	lea    rdi,[rsp+0x30]
=> 0x0000555555559a71 <+97>:	call   0x555555559370 <core::ptr::drop_in_place>
   0x0000555555559a76 <+102>:	jmp    0x555555559a78 <ddd::main+104>
   0x0000555555559a78 <+104>:	mov    rdi,rsp
   0x0000555555559a7b <+107>:	call   0x555555559370 <core::ptr::drop_in_place>
   0x0000555555559a80 <+112>:	add    rsp,0x78
   0x0000555555559a84 <+116>:	ret
   0x0000555555559a85 <+117>:	mov    rdi,rsp
   0x0000555555559a88 <+120>:	call   0x555555559370 <core::ptr::drop_in_place>
   0x0000555555559a8d <+125>:	mov    rdi,QWORD PTR [rsp+0x68]
   0x0000555555559a92 <+130>:	call   0x555555559050 <_Unwind_Resume@plt>
   0x0000555555559a97 <+135>:	ud2
   0x0000555555559a99 <+137>:	mov    QWORD PTR [rsp+0x68],rax
   0x0000555555559a9e <+142>:	mov    DWORD PTR [rsp+0x70],edx
   0x0000555555559aa2 <+146>:	jmp    0x555555559a85 <ddd::main+117>
End of assembler dump.

(gdb) x/xg $rdi
0x7fffffffe3c0: 0x00005555555a09f0

(gdb) x/4bx 0x00005555555a09f0
0x5555555a09f0: 0x4a  0x6f  0x68  0x6e    ; John
```

通过反汇编结果，可以看出首先释放的是 `john` 变量。

&nbsp;

### Deref 和 DerefMut

为了能够像普通指针一样能够解引用被指向类型的调用方法，智能指针类型通常实现 `Deref` 特征，这允许用户对这些类型使用 **解引用** 运算符`*`。虽然 `Deref` 只为你提供了只读权限，但是还有 `DerefMut`，它可以为你提供对底层类型的可变引用。

`Deref`具有以下类型签名:

```rust
// library/core/src/ops/deref.rs

pub trait Deref {
  type Target: ?Sized;
  
  fn deref(&self) -> &Self::Target;
}
```

&nbsp;

它定义了一个名为 `deref` 的方法，并通过引用获取 `self` 参数，然后返回对底层类型的不可变引用。与Rust的 `deref` 强制性特征结合，能够大幅减少开发编写代码的工作量。`deref` 强制性特征是指类型自动从一种类型的引用转换成另一种类型的其它引用。

```rust
use std::ops::Deref;

struct DerefExample<T> {
    value: T
}

impl<T> Deref for DerefExample<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.value
    }
}

fn main() {
    let x = DerefExample { value: 'a' };
    assert_eq!('a', *x);
}
```

&nbsp;

## 智能指针分类

* `Box<T>`: 在堆上分配值
* `Rc<T>`: 引用计数，可以有多个拥有者
* `Arc<T>`: 用于原子性引用计数(保证多线程的安全性)
* `Cell<T>`: 可以获得多个可变引用
* `RefCell<T>`: 以不可变引用修改值

&nbsp;

### Box

在堆(heap)上为值分配内存，栈上只是一个指向该内存的指针。

* 单一所有者。
* 运行时才知道所需内存大小。
* 转移所有权时，不想复制数据。

```rust

   stack              heap
  +=====+           +=======+
  | ptr | --------> | value |    Box<T>
  +=====+           +=======+
```

&nbsp;

和引用一致的使用方式，没有特殊功能。超出作用域自动释放。

```rust
fn main() {
  // 在堆上分配 sizeof(i32) 大小内存，并存储值 5。
  let mut p = Box::new(5);
    
  *p += 1;
  let x: i32 = *p;
    
  assert_eq!(x, 6);
}
```

```x86asm
0x0000555555559452 <+18>:	mov    eax,0x4
0x0000555555559457 <+23>:	mov    rdi,rax
0x000055555555945a <+26>:	mov    rsi,rax
0x000055555555945d <+29>:	call   0x555555559730 <alloc::alloc::exchange_malloc>
0x0000555555559462 <+34>:	mov    rcx,rax
0x0000555555559465 <+37>:	mov    DWORD PTR [rax],0x5

; let mut p = Box::new(5);
0x000055555555946b <+43>:	mov    QWORD PTR [rsp+0x60],rcx     // p(stack) -> heap -> 0x00000005

(gdb) info locals
x = 6
p = 0x55555559c9d0

(gdb) x/xw 0x55555559c9d0                                       // 在heap上为值分配内存
0x55555559c9d0:	0x00000006

(gdb) info proc mappings
0x55555559c000     0x5555555bd000    0x21000        0x0 [heap]  // heap内存范围

(gdb) disass                                                    // 超出作用域，所有权自动释放
0x0000555555559630 <+496>:	lea    rdi,[rsp+0x60]
0x0000555555559635 <+501>:	call   0x555555559230 <core::ptr::drop_in_place>
0x000055555555963a <+506>:	mov    rdi,QWORD PTR [rsp+0xf0]
```

#### Box创建数组类型

```rust
fn main() {
    // 以数组实现类 calloc 功能。
    let mut p = Box::new([0i64; 10]);

    p[1] = 10;
    p[2] = 20;

    println!("{:?} {:?}", p[2], p);
}
```

```x86asm
(gdb) info locals                                               // 数组值被分配在堆上
p = 0x5555555a09d0

(gdb) x/10xg 0x5555555a09d0                                     // heap上值内容
0x5555555a09d0:	0x0000000000000000	0x000000000000000a
0x5555555a09e0:	0x0000000000000014	0x0000000000000000
0x5555555a09f0:	0x0000000000000000	0x0000000000000000
0x5555555a0a00:	0x0000000000000000	0x0000000000000000
0x5555555a0a10:	0x0000000000000000	0x0000000000000000
```

#### Box类型所有权

`Box`类型的所有权语义取决于包装类型。如果基础类型为 `Copy`，那么 `Box`实例将成为副本，否则默认情况下发生**所有权转移**。

```rust
fn box_ref<T>(b: T) -> Box<T> {
  // 实例副本
  let a = b;
  Box::new(a)
}

struct Foo(i32);

fn main() {
  let boxed_one = Box::new(Foo(0x64));
  let unboxed_one = *boxed_one;
  box_ref(unboxed_one);
}
```

```x86asm
; 进程内存地址映射表
(gdb) info proc  mappings
process 1883
Mapped address spaces:

          Start Addr           End Addr       Size     Offset objfile
      0x555555554000     0x555555559000     0x5000        0x0 /root/rs/ddd/target/debug/ddd
      0x555555559000     0x55555558b000    0x32000     0x5000 /root/rs/ddd/target/debug/ddd
      0x55555558b000     0x555555598000     0xd000    0x37000 /root/rs/ddd/target/debug/ddd
      0x555555598000     0x55555559b000     0x3000    0x43000 /root/rs/ddd/target/debug/ddd
      0x55555559b000     0x55555559c000     0x1000    0x46000 /root/rs/ddd/target/debug/ddd
      0x55555559c000     0x5555555bd000    0x21000        0x0 [heap]

      ...

      0x7ffffffde000     0x7ffffffff000    0x21000        0x0 [stack]      

(gdb) info locals
boxed_one = 0x55555559c9d0

; 变量boxed_one指向heap地址: 0x55555559c9d0位置，内容为0x64
(gdb) x/xg 0x55555559c9d0
0x55555559c9d0:	0x0000000000000064

; let unboxed_one = *boxed_one 解引用，获取Foo struct数据
(gdb) info locals
unboxed_one = ddd::Foo (
  100
)

; fn box_ref<T>(b: T) -> Box<T>
(gdb) disassemble
Dump of assembler code for function ddd::box_ref:
=> 0x0000555555559ad0 <+0>:	sub    rsp,0x18
   0x0000555555559ad4 <+4>:	mov    DWORD PTR [rsp+0xc],edi
   0x0000555555559ad8 <+8>:	mov    DWORD PTR [rsp+0x10],edi
   0x0000555555559adc <+12>:	mov    DWORD PTR [rsp+0x14],edi
   0x0000555555559ae0 <+16>:	mov    eax,0x4
   0x0000555555559ae5 <+21>:	mov    DWORD PTR [rsp+0x8],edi
   0x0000555555559ae9 <+25>:	mov    rdi,rax
   0x0000555555559aec <+28>:	mov    rsi,rax
   0x0000555555559aef <+31>:	call   0x555555559660 <alloc::alloc::exchange_malloc>
   0x0000555555559af4 <+36>:	mov    rcx,rax
   0x0000555555559af7 <+39>:	mov    edx,DWORD PTR [rsp+0x8]
   0x0000555555559afb <+43>:	mov    DWORD PTR [rax],edx
   0x0000555555559afd <+45>:	mov    QWORD PTR [rsp],rcx
   0x0000555555559b01 <+49>:	mov    rax,QWORD PTR [rsp]
   0x0000555555559b05 <+53>:	add    rsp,0x18
   0x0000555555559b09 <+57>:	ret
End of assembler dump.

; main函数调用box_ref，通过$rdi寄存器进行传递
; 参数按值传递
(gdb) p/x $edi
$2 = 0x64

; Box::new(a) 返回堆上地址，该内存空间存储具体数据
(gdb) p/x $rcx
$4 = 0x55555559c9f0

(gdb) x/xg 0x55555559c9f0
0x55555559c9f0:	0x0000000000000064

; box_ref返回(rax 堆上地址) 
0x0000555555559b01 <+49>:	mov    rax,QWORD PTR [rsp]
0x0000555555559b05 <+53>:	add    rsp,0x18
0x0000555555559b09 <+57>:	ret

; 所有权释放
0x0000555555559313 <+67>:	call   0x555555559ad0 <ddd::box_ref>
0x0000555555559318 <+72>:	mov    QWORD PTR [rsp],rax
0x000055555555931c <+76>:	jmp    0x55555555931e <ddd::main+78>
0x000055555555931e <+78>:	mov    rax,QWORD PTR [rsp]
0x0000555555559322 <+82>:	mov    QWORD PTR [rsp+0x20],rax
0x0000555555559327 <+87>:	lea    rdi,[rsp+0x20]
0x000055555555932c <+92>:	call   0x5555555591e0 <core::ptr::drop_in_place>
0x0000555555559331 <+97>:	jmp    0x555555559333 <ddd::main+99>

; 释放heap 内存
0x0000555555559333 <+99>:	mov    rdi,QWORD PTR [rsp+0x10]
0x0000555555559338 <+104>:	call   0x555555559d10 <alloc::alloc::box_free>
0x000055555555933d <+109>:	add    rsp,0x48
0x0000555555559341 <+113>:	ret

(gdb) x/xg $rsp+0x10
0x7fffffffe3d0:	0x000055555559c9d0

(gdb) x/xg 0x000055555559c9d0
0x55555559c9d0:	0x0000000000000064
```

&nbsp;

#### 递归类型定义

`Box` 类型适用于创建递归类型定义。

```rust
struct Node {
  data: u32,
  next: Option<Node>
}

fn main() {
  let a = Node { data: 33, next: None };
}
```

&nbsp;

在编译上述代码时，我们得到一下错误提示: 

```rust
root@8d75790f92f5:~/rs/ddd/src# cargo c
    Checking ddd v0.1.0 (/root/rs/ddd)
error[E0308]: mismatched types
  --> src/main.rs:12:36
   |
12 |     let a = Node { data: 33, next: None };
   |                                    ^^^^ expected struct `Box`, found enum `Option`
   |
   = note: expected struct `Box<Node>`
                found enum `Option<_>`
```

由于每个数据片段在编译时都需要确定静态的已知尺寸，因此在 `Rust` 中这是一种不可表示的类型。我们需要让 `next` 字段具有固定大小，可以通过 `next` 放在指针后面来实现，因为指针总是具有固定大小。可以使用 `Box` 类型修改 `Node`结构体的定义:

```rust
struct Node {
  data: u32,
  next: Option<Box<Node>>
}
```

当定义需要隐藏在不定长结构后面的递归类型时，也可以使用 `Box` 类型。

## 引用计数的智能指针

引用计数类型支持某个粒度级别的垃圾回收。在这种方法中，智能指针类型允许用户对包装值进行多次引用。在内部，智能指针使用引用计数器(这里是refcount)来统计自发放的并且活动的引用数量，不过它只是一个整数值。当引用包装的智能指针值的变量超出作用域时，`refcount`的值就会递减。一旦该对象的所有权引用都消失，`refcount` 的值也会变成0， 之后该值被销毁。这就是引用计数指针常见的工作模式。

Rust提供了两种引用计数指针类型。

* `Rc<T>` : 用于单线程环境
* `Arc<T>`: 用于多线程环境

### Rc

引用计数(reference counting)启用多所有权。当引用数为 `0` 时，进行清理。

* 堆分配内存，共享所有权
* 单线程，使用非原子计数
* 不可变引用，无法修改值
* 降级获取没有所有权的弱指针(weak pointer)
* 存在循环引用无法释放的问题(用弱指针解决)


`RC`底层数据结构: 

```rust

   stack              heap
  +=====+           +============+
  | ptr | --------> | strong_cnt |    Rc<T>
  +=====+           +------------+
                    | weak_cnt   |
                    +------------+
                    | value      |
                    +============+
```

```rust
// library/alloc/src/rc.rs
struct RcBox<T: ?Sized> {
    strong: Cell<usize>,            // 强引用计数
    weak: Cell<usize>,              // 弱引用计数
    value: T,                       // 值
}
```

当我们与一个 `Rc`类型交互时，其内部会发生如下变化。

* 通过调用 `Clone()` 获取对 `Rc` 的一个新共享引用时，`Rc`会增加其内部引用计数。`Rc`内部使用 `Cell`类型处理其引用计数。
* 当引用超出作用域时，它会对引用计数器执行递减操作。
* 当所有共享引用计数超出作用域时，`refcount` 会变成0。此时，`Rc`上最后一次 `drop` 调用会执行相关的资源清理工作。

`Rc<T>` 主要通过两种方式使用:

* 静态方法 `Rc::new` 会生成一个新的引用计数器。
* `clone` 方法会增加强引用计数并分发一个新的 `Rc<T>`。

```rust
use std::rc::Rc;

fn main() {
    let rc = Rc::new(5);
    assert_eq!(Rc::strong_count(&rc), 1);

    {
        // 增加引用，计数 +1
        let rc2 = rc.clone();
        assert_eq!(Rc::strong_count(&rc), 2);

        // 解引用
        assert_eq!(*rc2, 5);
    }   // 超出作用域，计数 -1

    assert_eq!(Rc::strong_count(&rc), 1);
}
```

```x86asm
// heap上存储的是RcBox 数据结构
(gdb) info locals
rc = alloc::rc::Rc<i32> {
  ptr: core::ptr::non_null::NonNull<alloc::rc::RcBox<i32>> {
    pointer: 0x55555559d9d0
  },
  phantom: core::marker::PhantomData<alloc::rc::RcBox<i32>>
}

(gdb) x/3xg 0x55555559d9d0
0x55555559d9d0:	0x0000000000000001	0x0000000000000001
0x55555559d9e0:	0x0000000000000005

// Rc::new执行后，rc引用计数为1
(gdb) x/xg 0x55555559f9d0
0x55555559f9d0:	0x0000000000000001

// rc.clone执行后，rc引用计数自加1, 即2
(gdb) c
Continuing.
Breakpoint 2, ddd::main () at /root/rs/ddd/src/main.rs:13
13	        assert_eq!(Rc::strong_count(&rc), 2);

(gdb) x/xg 0x55555559f9d0
0x55555559f9d0:	0x0000000000000002

// 离开rc2作用域，计数器自减1，即1
(gdb) c
Continuing.
Breakpoint 3, ddd::main () at /root/rs/ddd/src/main.rs:18
18	    assert_eq!(Rc::strong_count(&rc), 1);

(gdb) x/xg 0x55555559f9d0
0x55555559f9d0:	0x0000000000000001
```

&nbsp;

`Rc` 内部会保留两种引用: **强引用(`Rc<T>`)** 和 **弱引用(`Weak<T>`)**。二者都会维护每种类型的引用数量的计数，但是仅在**强引用计数值为零时，才会释放该值**。这样做的目的是数据结构的实现可能需要多次指向同一事物。

&nbsp;

#### 弱引用

```rust
use std::rc::{Rc, Weak};

fn main() {
    let wr: Weak<i32>;

    {
        let rc = Rc::new(5);

        // 降级, 生成弱引用
        wr = Rc::downgrade(&rc);

        // 不影响强引用计数
        assert_eq!(Rc::strong_count(&rc), 1);
        assert_eq!(Rc::weak_count(&rc), 1);

        // 弱引用不能保证目标值存活，所以不能直接解引用。
        // 升级成强引用再操作。如已释放，返回None。
        if let Some(rc2) = wr.upgrade() {
            assert_eq!(*rc2, 5);
        } else {
            panic!("upgrade: None");
        }
    }   // Rc drop!!!

    assert_eq!(wr.upgrade(), None);
}
```

```x86asm
(gdb) info locals
rc = alloc::rc::Rc<i32> {
  ptr: core::ptr::non_null::NonNull<alloc::rc::RcBox<i32>> {
    pointer: 0x5555555a09d0
  },
  phantom: core::marker::PhantomData<alloc::rc::RcBox<i32>>
}

; let rc = Rc::new(5) 引用计数置为1
(gdb) x/xg 0x5555555a09d0
0x5555555a09d0:	0x0000000000000001

; wr = Rc::downgrade(&rc), rc降级为弱引用
; 引用计数不受影响
(gdb) c
Continuing.
Breakpoint 2, ddd::main () at /root/rs/ddd/src/main.rs:15
15	        assert_eq!(Rc::strong_count(&rc), 1);

; 弱引用计数器为1
rc = Rc(strong=1, weak=1) = {
  value = 5,
  strong = 1,
  weak = 1
}

(gdb) x/xg 0x5555555a09d0
0x5555555a09d0:	0x0000000000000001

; wr.upgrade(), wr升级为强引用
; 引用计数自增1
(gdb) c
Continuing.
Breakpoint 3, ddd::main () at /root/rs/ddd/src/main.rs:19
19	            assert_eq!(*rc2, 5);

(gdb) x/xg 0x5555555a09d0
0x5555555a09d0:	0x0000000000000002

; 超出作用域, 引用计数置为0
(gdb) c
Continuing.
Breakpoint 4, ddd::main () at /root/rs/ddd/src/main.rs:25
25	    assert_eq!(wr.upgrade(), None);

(gdb) x/xg 0x5555555a09d0
0x5555555a09d0:	0x0000000000000000
```

> gdb反汇编调试尽可能使用rust-gdb，而不要使用gdb。因为，rust内部封装结构是否构成弱引用必然在实际内存数据上有所体现，但未必暴露给我们这些外部用户。所以gdb调试结果和rust-gdb调试结果有所出入。

&nbsp;

#### 循环引用问题

可以使用弱引用打破引用循环。链表可以通过将引用计数分别指向下一个元素和上一个元素的方式维护链接。更好的方法是 **对一个方向使用强引用，而对另一个方向使用弱引用** 。

##### 单链表实例

```rust
use std::rc::Rc;

#[derive(Debug)]
struct LinkedList<T> {
    head: Option<Rc<Node<T>>>
}

#[derive(Debug)]
struct Node<T> {
    next: Option<Rc<Node<T>>>,
    data: T
}

impl<T> LinkedList<T> {
    fn new() -> Self {
        LinkedList { head: None }
    }

    fn append(&self, data: T) -> Self {
        LinkedList {
            head: Some(Rc::new(Node {
                data: data,
                next: self.head.clone()
            }))
        }
    }
}

fn main() {
    let list_of_nums = LinkedList::new().append(1).append(2);
    println!("numbs: {:?}", list_of_nums);

    let list_of_strs = LinkedList::new().append("foo").append("bar");
    println!("strs: {:?}", list_of_strs);
}
```

&nbsp;

反汇编单向链表:

```x86asm
; list_of_nums
(gdb) info locals
list_of_nums = ddd::LinkedList<i32> {
  head: core::option::Option<alloc::rc::Rc<ddd::Node<i32>>>::Some(alloc::rc::Rc<ddd::Node<i32>> {
      ptr: core::ptr::non_null::NonNull<alloc::rc::RcBox<ddd::Node<i32>>> {
        pointer: 0x5555555a29d0
      },
      phantom: core::marker::PhantomData<alloc::rc::RcBox<ddd::Node<i32>>>
    })
}
(gdb) x/4xg 0x5555555a29d0
0x5555555a29d0:	0x0000000000000001	0x0000000000000001
0x5555555a29e0:	0x00005555555a2910	0x0000000000000002    ; 0x00005555555a2910 下一个链表节点地址 

(gdb) x/4xg 0x00005555555a2910
0x5555555a2910:	0x0000000000000001	0x0000000000000001
0x5555555a2920:	0x0000000000000000	0x0000000000000001

; list_of_strs
(gdb) info locals
list_of_strs = ddd::LinkedList<&str> {
  head: core::option::Option<alloc::rc::Rc<ddd::Node<&str>>>::Some(alloc::rc::Rc<ddd::Node<&str>> {
      ptr: core::ptr::non_null::NonNull<alloc::rc::RcBox<ddd::Node<&str>>> {
        pointer: 0x5555555a2aa0
      },
      phantom: core::marker::PhantomData<alloc::rc::RcBox<ddd::Node<&str>>>
    })
}

(gdb) x/4xg 0x5555555a2aa0
0x5555555a2aa0:	0x0000000000000001	0x0000000000000001
0x5555555a2ab0:	0x00005555555a2a70	0x000055555558f00b

(gdb) x/3xb 0x000055555558f00b
0x55555558f00b:	0x62	0x61	0x72                        ; bar

(gdb) x/4xg 0x00005555555a2a70
0x5555555a2a70:	0x0000000000000001	0x0000000000000001
0x5555555a2a80:	0x0000000000000000	0x000055555558f008

(gdb) x/3xb 0x000055555558f008
0x55555558f008:	0x66	0x6f	0x6f                        ; foo
```

&nbsp;

##### 双向链表实例

可以使用 `downgrade` 方法将一个 `Rc<T>` 类型转换成一个 `Weak<T>`类型。类似地，可以使用 `upgrade` 方法将一个 `Weak<T>` 类型转换成一个 `R<T>` 类型。`downgrade` 方法始终有效，而在弱引用上调用 `upgrade` 方法时，实际的值可能已经被删除，在这种情况下，你将获得的值是`None`。 所以，添加一个只想上一个节点的弱指针。

```rust
use std::rc::Rc;
use std::rc::Weak;
use std::cell::RefCell;

#[derive(Debug)]
struct LinkedList<T> {
    head: Option<Rc<LinkedListNode<T>>>
}

#[derive(Debug)]
struct LinkedListNode<T> {
    next: Option<Rc<LinkedListNode<T>>>,
    prev: RefCell<Option<Weak<LinkedListNode<T>>>>,
    data: T
}

impl<T> LinkedList<T> {
    fn new() -> Self {
        LinkedList { head: None }
    }

    fn append(&mut self, data: T) -> Self {
        let new_node = Rc::new(LinkedListNode {
            data: data,
            next: self.head.clone(),
            prev: RefCell::new(None)
        });

        match self.head.clone() {
            Some(node) => {
                let mut prev = node.prev.borrow_mut();
                *prev = Some(Rc::downgrade(&new_node));
            },
            None => {

            }
        }

        LinkedList {
            head: Some(new_node)
        }
    }
}

fn main() {
    let list_of_nums = LinkedList::new().append(1).append(2).append(3);
    println!("nums: {:?}", list_of_nums);
}
```

双向链表结果:

```rust
list_of_nums = ddd::LinkedList<i32> {
  head: core::option::Option<alloc::rc::Rc<ddd::LinkedListNode<i32>>>::Some(Rc(strong=1, weak=1) = {
      value = ddd::LinkedListNode<i32> {
        next: core::option::Option<alloc::rc::Rc<ddd::LinkedListNode<i32>>>::Some(Rc(strong=1, weak=1) = {
            value = ddd::LinkedListNode<i32> {
              next: core::option::Option<alloc::rc::Rc<ddd::LinkedListNode<i32>>>::Some(Rc(strong=1, weak=0) = {
                  value = ddd::LinkedListNode<i32> {
                    next: core::option::Option<alloc::rc::Rc<ddd::LinkedListNode<i32>>>::None,
                    prev: RefCell(borrow=0) = {
                      value = core::option::Option<alloc::rc::Weak<ddd::LinkedListNode<i32>>>::Some(alloc::rc::Weak<ddd::LinkedListNode<i32>> {
                          ptr: core::ptr::non_null::NonNull<alloc::rc::RcBox<ddd::LinkedListNode<i32>>> {
                            pointer: 0x5555555a3a10
                          }
                        }),
                      borrow = 0
                    },
                    data: 1
                  },
                  strong = 1,
                  weak = 0
                }),
              prev: RefCell(borrow=0) = {
                value = core::option::Option<alloc::rc::Weak<ddd::LinkedListNode<i32>>>::Some(alloc::rc::Weak<ddd::LinkedListNode<i32>> {
                    ptr: core::ptr::non_null::NonNull<alloc::rc::RcBox<ddd::LinkedListNode<i32>>> {
                      pointer: 0x5555555a3a50
                    }
                  }),
                borrow = 0
              },
              data: 2
            },
            strong = 1,
            weak = 1
          }),
        prev: RefCell(borrow=0) = {
          value = core::option::Option<alloc::rc::Weak<ddd::LinkedListNode<i32>>>::None,
          borrow = 0
        },
        data: 3
      },
      strong = 1,
      weak = 1
    })
}
```

&nbsp;

只让结构体的某个部分可变，可以通过RefCell进行修改:

```rust
// 1. 添加RefCell引用
use std::cell::RefCell;

// 2. 将RefCell添加到LinkedListNode
#[derive(Debug)]
struct LinkedListNode<T> {
  next: Option<Rc<LinkedListNode<T>>>,
  prev: RefCell<Option<Weak<LinkedListNode<T>>>>,
  data: T
}

// 3. 修改append方法以创建新的RefCell，并通过RefCell可变借用更新之前的引用
fn append(&mut self, data: T) -> Self {
  let new_node = Rc::new(LinkedListNode{
    data: data,
    next: self.head.Clone(),
    prev: RefCell::new(None)
  });

  match self.head.Clone() {
    Some(node) => {
      let mut prev = node.prev.borrow_mut();    // 借用
      *prev = Some(Rc::downgrade(&new_node));
    },
    None => {
    }
  }

  LinkedList {
    head: Some(new_node)
  }
}
```

&nbsp;

#### 自动解引用

自动解引用，可直接调用值方法。

```rust
use std::rc::Rc;

struct User {
    age: u8
}

impl User {
    fn test(&self) {
        println!("{}", self.age);
    }
}

fn main() {
    let rc = Rc::new(User{ age: 10 });
    rc.test();
}
```

&nbsp;

## 内部可变性

Rust通过在任何给定作用域中仅允许一个可变引用，从而在编译时保证我们免受指针别名的影响。但是，在某些情况下，它会变得非常严格，因为严格的借用检查使我们知道由于代码的安全性而不能通过编译器的编译。对于这种情况，其中一个解决方案是将借用检查从编译时移动到运行时，这是通过内部可变性实现的。

  * 继承可变性: 是获得某些结构体的 `&mut` 引用时默认取得的可变性。意味着你可以修改结构体中的任意字段。
  * 内部可变性: 这种可变性中，即使你有一个引用某种类型的 `&SomeStruct`，如果其中的字段类型为 `Cell<T>` 或 `RefCell<T>`, 那么仍然可以修改其字段。

&nbsp;

**内部可变性**(interior mutability)利用unsafe绕过借用规则，允许通过不可变引用修改内部值。

  * 唯一所有权，运行期借用规则检查。
  * 运行时行为，出错`panic!`。
  * 单线程。

  > 编译器静态规则检查过于保守，某些设计需要运行期操作。 

&nbsp;

在运行时不存在两个可变借用。这些类型将多个可变借用的检测从编译时移动到了运行时，如果存在对值的两个可变借用，就会发生异常。当希望向用户公开不可变API时，通常会遇到内部可变性，不过上述API内部存在部分可变性。标准库中有两个通用的智能指针类型提供了共享可变性: `Cell` 和 `RefCell`。

&nbsp;

### Cell

`Cell<T>`类型是一种智能指针类型，可以为值提供可变性，甚至允许值位于不可引用之后。它以极低的开销提供了此功能，并具有最简单的API。

&nbsp;

#### Cell优势?

便捷版本`Cell`，开辟一块儿始终可变内存区域。

* 栈内存分配。
* 仅适合实现`Copy`特征的类型。

```rust
use std::cell::Cell;

struct Data {
    x: i64,
    y: Cell<i64>,
}

fn main() {
    let d = Data{ x: 1, y: Cell::new(2) };

    // d.x = 10;
    // ^^^^^^^^ cannot assign

    d.y.set(200);
    assert_eq!(d.y.get(), 200);
}
```

```x86asm
(gdb) ptype d.y                         // Cell数据类型
type = struct core::cell::Cell<i64> {
  value: core::cell::UnsafeCell<i64>,
}

(gdb) info locals
d = ddd::Data {
  x: 1,
  y: core::cell::Cell<i64> {
    value: core::cell::UnsafeCell<i64> {
      value: 2
    }
  }
}

(gdb) p/x &d                            // 分配在stack
$1 = 0x7fffffffe350

(gdb) x/2xg 0x7fffffffe350
0x7fffffffe350:	0x0000000000000001	0x0000000000000002

(gdb) info proc  mappings
0x7ffffffde000     0x7ffffffff000    0x21000        0x0 [stack]
```

```x86asm
// d.y.set(200) 实现

0x0000555555559585 <+53>:	mov    rdi,rcx
0x0000555555559588 <+56>:	mov    esi,0xc8
0x000055555555958d <+61>:	call   0x5555555591b0 <core::cell::Cell<T>::set>

(gdb) p/x $rsp+0x40
$1 = 0x7fffffffe350

(gdb) x/2xg  0x7fffffffe350
0x7fffffffe350:	0x0000000000000001	0x00000000000000c8          // 0xc8 被设置为新值
```

```rust
// refCell.set() 函数源码剖析
pub fn set(&self, val: T) {
    let old = self.replace(val);    // 设置新值
    drop(old);                      // 删除旧值
}
```

&nbsp;

#### Cell 常用方法

* `Cell::new`: 允许你通过传递任意类型 `T` 来创建 `Cell` 类型的新实例。
* `get`: 允许你复制单元(cell)中的值。仅当包装类型 `T` 为**Copy**时，该方法才有效。
* `set`: 允许用户修改内部的值，即使该值位于某个不可变引用的后面。

以下实例将两个可变引用修改bag中的内容:

```rust
use std::cell::Cell;

#[derive(Debug)]
struct Bag {
    item: Box<u32>
}

fn main() {
    let mut bag = Cell::new(Bag { item: Box::new(1) });
    let hand1 = &mut bag;
    let hand2 = &mut bag;

    *hand1 = Cell::new(Bag { item: Box::new(2) });

}
```

&nbsp;

由于借用规则限制，不支持同时多次修改两个可变借用。

```rust
error[E0499]: cannot borrow `bag` as mutable more than once at a time
  --> src/main.rs:15:17
   |
14 |     let hand1 = &mut bag;
   |                 -------- first mutable borrow occurs here
15 |     let hand2 = &mut bag;
   |                 ^^^^^^^^ second mutable borrow occurs here
16 |
17 |     *hand1 = Cell::new(Bag { item: Box::new(2) });
   |     ------ first borrow later used here

error: aborting due to previous error
```

&nbsp;

可以通过将bag的值封装到Cell中来让它正常运转。

```rust
use std::cell::Cell;

#[derive(Debug)]
struct Bag {
    item: Box<u32>
}

fn main() {
    let bag = Cell::new(Bag { item: Box::new(1) });
    let hand1 = &bag;
    let hand2 = &bag;

    hand1.set(Bag { item: Box::new(2) });
    hand2.set(Bag { item: Box::new(3) });
}
```

&nbsp;

### RefCell

 如果需要某个非**Copy**类型支持**Cell**的功能，可以使用**RefCell**类型。

 它采用了和借用类似的读/写模式，但是将借用检查移动到运行时，这很方便，但不是零成本。`RefCell` 分发值的引用不像 `Cell`类型那样按值返回。

#### RefCell 实例1: 基本使用

```rust
use std::cell::RefCell;

fn main() {
    let c = RefCell::new(5);

    {
        let r1 = c.borrow();            // &T
        let r2 = c.borrow();            // &T

        assert_eq!(*r1, 5);
        assert_eq!(*r1, *r2);
    }

    {
        let mut r = c.borrow_mut();     // &mut T
        *r = 10;
    }

    assert_eq!(c.into_inner(), 10);     // c 被释放
}
```

```x86asm
// RefCell 结构

(gdb) ptype c
type = struct core::cell::RefCell<i32> {
  borrow: core::cell::Cell<isize>,
  value: core::cell::UnsafeCell<i32>,
}

// 结构体示意图

   stack              heap
  +=====+           +============+
  | ptr | --------> | borrow_cnt |    RefCell<T>
  +=====+           +------------+    borrow_mut: count = -1
                    | value      |
                    +============+
                    
```

```x86asm
// RefCell::new(5) 初始化后，borrow计数为0

(gdb) info locals
c = core::cell::RefCell<i32> {
  borrow: core::cell::Cell<isize> {
    value: core::cell::UnsafeCell<isize> {
      value: 0
    }
  },
  value: core::cell::UnsafeCell<i32> {
    value: 5
  }
}

(gdb) x/2xg 0x7fffffffe1d0
0x7fffffffe1d0:	0x0000000000000000	0x0000000000000005


// borrow_mut调用borrow_cnt()后，将计数置为-1

c = core::cell::RefCell<i32> {
  borrow: core::cell::Cell<isize> {
    value: core::cell::UnsafeCell<isize> {
      value: -1
    }
  },
  value: core::cell::UnsafeCell<i32> {
    value: 5
  }
}

// 调用borrow_cnt后，结构体发生改变
// let mut r = c.borrow_mut();

(gdb) ptype r
type = struct core::cell::RefMut<i32> {
  value: *mut i32,
  borrow: core::cell::BorrowRefMut,
}

// let mut r = c.borrow_mut(); 后，r变量引用置为负数，可通过r进行修改r指向堆上的值
r = core::cell::RefMut<i32> {
  value: 0x7fffffffe1d8,
  borrow: core::cell::BorrowRefMut {
    borrow: 0x7fffffffe1d0
  }
}

(gdb) x/xg 0x7fffffffe1d8                       // 值
0x7fffffffe1d8:	0x0000000000000005

(gdb) x/xg 0x7fffffffe1d0                       // 一个负数补码
0x7fffffffe1d0:	0xffffffffffffffff
```

&nbsp;

#### 实例2: 改造Rc

改造`Rc`，使其可变。

```rust
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let rc = Rc::new(RefCell::new(5));
    let rc2 = rc.clone();

    {
        let mut r = rc.borrow_mut();
        *r = 100;
    }

    assert_eq!(*rc2.borrow(), 100);
}
```

```x86asm
(gdb) info locals
r = core::cell::RefMut<i32> {
  value: 0x55555559f928,
  borrow: core::cell::BorrowRefMut {
    borrow: 0x55555559f920
  }
}

rc2 = alloc::rc::Rc<core::cell::RefCell<i32>> {
  ptr: core::ptr::non_null::NonNull<alloc::rc::RcBox<core::cell::RefCell<i32>>> {
    pointer: 0x55555559f910
  },
  phantom: core::marker::PhantomData<alloc::rc::RcBox<core::cell::RefCell<i32>>>
}

rc = alloc::rc::Rc<core::cell::RefCell<i32>> {
  ptr: core::ptr::non_null::NonNull<alloc::rc::RcBox<core::cell::RefCell<i32>>> {
    pointer: 0x55555559f910
  },
  phantom: core::marker::PhantomData<alloc::rc::RcBox<core::cell::RefCell<i32>>>
}

// let mut r = rc.borrow_mut(), r borrow指向rc2(rc) RefCell.value地址
(gdb) x/4xg 0x55555559f910
0x55555559f910:	0x0000000000000002	0x0000000000000001
0x55555559f920:	0xffffffffffffffff	0x0000000000000005

// *r = 100, 更改r.borrow 指向heap内容，即rc2(rc).value位置的值
(gdb) x/4xg 0x55555559f910
0x55555559f910:	0x0000000000000002	0x0000000000000001
0x55555559f920:	0xffffffffffffffff	0x0000000000000064
```

&nbsp;

#### 实例3: borrow_mut 和 borrow使用实例

```rust
use std::cell::RefCell;

#[derive(Debug)]
struct Bag {
    item: Box<u32>
}

fn main() {
    let bag = RefCell::new(Bag { item: Box::new(1) });
    let hand1 = &bag;
    let hand2 = &bag;
    *hand1.borrow_mut() = Bag { item: Box::new(2) };
    *hand2.borrow_mut() = Bag { item: Box::new(3) };
    let borrowed = hand1.borrow();
    println!("{:?}", borrowed);
}
```

&nbsp;

尝试在统一作用域中调用上述两种方法:

```rust
use std::cell::RefCell;

#[derive(Debug)]
struct Bag {
    item: Box<u32>
}

fn main() {
    let bag = RefCell::new(Bag { item: Box::new(1) });
    let hand1 = &bag;
    let hand2 = &bag;
    *hand1.borrow_mut() = Bag { item: Box::new(2) };
    *hand2.borrow_mut() = Bag { item: Box::new(3) };
    println!("{:?} {:?}", hand1.borrow(), hand1.borrow_mut());
}
```

```rust
root@8d75790f92f5:~/rs/ddd/src# cargo r
   Compiling ddd v0.1.0 (/root/rs/ddd)
    Finished dev [unoptimized + debuginfo] target(s) in 3.87s
     Running `/root/rs/ddd/target/debug/ddd`
thread 'main' panicked at 'already borrowed: BorrowMutError', src/main.rs:20:49
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

出现上述错误，是因为独占可变访问具有相同的所有权规则。但是，对于 `RefCell`，这会在运行时进行检查。对于这种情况，必须明确使用单独的代码块分隔借用，或者使用 `drop` 方法删除引用。

&nbsp;

### RefCell Vs. Cell

`Cell`比`RefCell` 更轻，性能更好，用法更方。但是`Cell`只能包装`Copy`类型，而`RefCell`可以包装任何类型，并且`RefCell`可以获取其内部包装对象的引用，并在运行时检测可变引用的唯一性。

具体数据结构:

```rust
// library/core/src/cell.rs

pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,                                                       // 内部对象
}
```

```rust
// RefCell
type BorrowFlag = isize;

pub struct RefCell<T: ?Sized> {
    borrow: Cell<BorrowFlag>,                                                   // 对象引用类别和计数器
    borrowed_at: Cell<Option<&'static crate::panic::Location<'static>>>,        // 借用最开始&活跃的位置
    value: UnsafeCell<T>,                                                       // 内部对象
}
```

```rust
// &mut T
pub struct RefMut<'b, T: ?Sized + 'b> {
    value: &'b mut T,
    borrow: BorrowRefMut<'b>,
}
```

`RefCell`内部维护了一个包装对象的引用计数，当`RefCell.borrow`获取一个共享引用时，内部引用计数加1，当获取的引用离开作用域时，内部引用计数减1，当`RefCell.borrow_mut`获取一个可变引用时，首先检测引用计数是否为0，如果为0，正常返回，否则panic；其实`RefCell.borrow`也会做类似的检测，当已经获取了可变引用也是直接panic，当然为了避免panic，我们可以用`RefCell.try_borrow`和`RefCell.try_borrow_mut`来获取一个`Result`类型。

因为`Cell`和`RefCell`两种类型都未实现`Sync` trait，所以这两种类型只能用于单线程中，不能跨线程操作，如果需要跨线程操作，就需要用到`Mutex`和`RwLock`了。

> Cell 和 RefCell类型不是线程安全(thead-safety)的。这意味着 `Rust` 不允许用户在多线程环境中共享这些类型。

### 内部可变性的应用

使用两个整数和sum方法来扩展结构体，以缓存求和的结果，并返回缓存的值。

```rust
use std::cell::Cell;

struct Point {
    x: u8,
    y: u8,
    cached_sum: Cell<Option<u8>>
}

impl Point {
    fn sum(&self) -> u8 {
        match self.cached_sum.get() {
            Some(sum) => {
                println!("Got from cache: {}", sum);
                sum
            },
            None => {
                let new_sum = self.x + self.y;
                self.cached_sum.set(Some(new_sum));
                println!("Set cache: {}", new_sum);
                new_sum
            }
        }
    }
}

fn main() {
    let p = Point { x: 8, y: 9 , cached_sum: Cell::new(None) };
    println!("Summed result: {}", p.sum());
    println!("Summed result: {}", p.sum());
}
```
