---
layout: post
title: 'rust所有权转移-引用和借用'
tags: [code]
---

# 引用和借用

引用(reference)不获取所有权，坚持单一所有者和单一职责，解决了共享访问障碍。按引用传递对象的方式称作借用(borrow)，这比转移所有权更有效。

```rust
fn main() {
    let a = 100;
    let r: &i32 = &a;          // &T: 不可变引用。

    assert_eq!(*r, a);
}
```

```rust
fn main() {
    let mut a = 100;
    let r: &mut i32 = &mut a;  // &mut T: 可变引用。
    *r += 1;

    assert_eq!(*r, 101);
}
```

&nbsp;

## 实现

引用的底层实现和指针基本一致，也就是说附加于引用身上的规则是编译器需要处理的。

两者区别在于：

1. **引用必须有效**。引用失效前，目标必须存活且不可改变。
2. 而指针未必，可能是空指针、野指针或悬垂指针。

```rust
fn main() {
    let mut a = 0;
    let _r = &mut a;
}
```

```rust
// /root/rs/closure/src/main.rs:
fn main() {
    0x0000555555559200 <+0>:    sub    rsp,0x10

    // let mut a = 0;
    0x0000555555559204 <+4>:    mov    DWORD PTR [rsp+0x4],0x0

    // let _r = &mut a;
    0x000055555555920c <+12>:   lea    rax,[rsp+0x4]                // 引用就是一个指针传递
    0x0000555555559211 <+17>:   mov    QWORD PTR [rsp+0x8],rax

    // }
=> 0x0000555555559216 <+22>:    add    rsp,0x10
   0x000055555555921a <+26>:    ret
End of assembler dump.
```

&nbsp;

引用不获取所有权，也不关心释放。

```rust
fn test(s: &mut String) {
    s.push_str("!!!") ;
}

fn main() {
    let mut s = String::from("hello");
    test(&mut s);
}
```

```rust
; /root/rs/closure/src/main.rs:
    // fn main() {
    0x000055555555ada0 <+0>:    sub    rsp,0x28
    0x000055555555ada4 <+4>:    lea    rax,[rip+0x332c1]        # 0x55555558e06c

    // let mut s = String::from("hello");
    0x000055555555adab <+11>:   mov    rdi,rsp
    0x000055555555adae <+14>:   mov    rsi,rax
    0x000055555555adb1 <+17>:   mov    edx,0x5
    0x000055555555adb6 <+22>:   call   0x55555555c8c0 <<alloc::string::String as core::convert::From<&str>>::from>
    0x000055555555adbb <+27>:   mov    rdi,rsp

    // test(&mut s);
 => 0x000055555555adbe <+30>:   call   0x55555555ad80 <closure::test>
    0x000055555555adc3 <+35>:   jmp    0x55555555adc5 <closure::main+37>

    // }
    0x000055555555adc5 <+37>:   mov    rdi,rsp
    0x000055555555adc8 <+40>:   call   0x55555555bc80 <core::ptr::drop_in_place>                ; 引用不获取所有权
    0x000055555555adcd <+45>:   add    rsp,0x28
    0x000055555555add1 <+49>:   ret
    0x000055555555add2 <+50>:   mov    rdi,rsp
    0x000055555555add5 <+53>:   call   0x55555555bc80 <core::ptr::drop_in_place>

    // fn main() {
    0x000055555555adda <+58>:   mov    rdi,QWORD PTR [rsp+0x18]
    0x000055555555addf <+63>:   call   0x555555559050 <_Unwind_Resume@plt>
    0x000055555555ade4 <+68>:   ud2
    0x000055555555ade6 <+70>:   mov    QWORD PTR [rsp+0x18],rax
    0x000055555555adeb <+75>:   mov    DWORD PTR [rsp+0x20],edx
    0x000055555555adef <+79>:   jmp    0x55555555add2 <closure::main+50>


// 作为s参数传入,并不会发生所有权转移

// Dump of assembler code for function closure::test:
// /root/rs/closure/src/main.rs:

//   fn test(s: &mut String) {
 => 0x000055555555ad80 <+0>:    push   rax
    0x000055555555ad81 <+1>:    lea    rax,[rip+0x332e1]        # 0x55555558e069
    0x000055555555ad88 <+8>:    mov    QWORD PTR [rsp],rdi

    // s.push_str("!!!");
    0x000055555555ad8c <+12>:   mov    rsi,rax
    0x000055555555ad8f <+15>:   mov    edx,0x3
    0x000055555555ad94 <+20>:   call   0x55555555c860 <alloc::string::String::push_str>

    // }
    0x000055555555ad99 <+25>:   pop    rax
    0x000055555555ad9a <+26>:   ret
```

&nbsp;

### 引用和指针区别

区分引用和原始指针(raw pointer)的区别。

引用声明也用 `&T`，指针 `*T` 。

```rust
fn main() {
    // reference
    let mut x = 100;
    let r : &mut i32 = &mut x;

    *r += 1;
    println!("{:?}", *r);

    // raw-pointer
    let mut y = 200;
    let p: *mut i32 = &mut y;

    unsafe {                        // 指针需要使用unsafe进行操作
        *p += 1;
        println!("{:?}", *p);
    }
}
```

&nbsp;

## 规则

引用与指针最大的区别在于附加以下的规则，这些需要编译器作出检查。

* 借用不得超过所有者有效范围(socpe)。
* 既定时间，只有一个可变引用(&mut T)，或只有多个不可变引用(&T)。
* 可变引用(&mut T)失效(超出范围或不再使用)前，**不能访问原变量**。
* 引用失效(超出范围或不再使用)前，**原变量冻结(freeze)，不可修改，转移或重新赋值**。

&nbsp;

可以按下面的逻辑理解：

* 只能有一个主人(owner)。
* 其他人可以租借：
    
    * 同一时刻，可以有多个人(含主人)读(&T)。
    * 同一时刻，只能有一个写(&mut T)，其它租客和主人都不能掺合。
    * 只要有租借，主人就不得修改、销毁(drop)、转卖(move)或更换(重赋值)。

值有效范围。

```rust
fn main() {
    let y: &i32;

    {
        let x = 5;
        y = &x;
            // ^^ borrowed value does not live long enough
    }
    // - `x` dropped here while still borrowed

    println!("{}", y);
}
```

&nbsp;

### 变量冻结

在借用未失效的情况下，改变原绑定，会导致不一致行为。

这与其他语言有很大的区别。

```rust
fn main() {
    let mut s = String::from("abc");
    let r = &s;
            // -- immutable borrow occurs here

     // 引用未失效，不能修改
    s.push_str("def");             
            // ^^^^^^^^^^^^^^^^^ mutable borrow occurs here

    println!("{:?}", r);
            // - immutable borrow later used here
}
```

```rust
fn main() {
    let mut a = 1;

    // 或 &mut a
    let r = &a;            
            // -- borrow of `a` occurs here

    // 引用未失效，不能重新赋值
    a = 50;                 
            // ^^^^^^ assignment to borrowed `a` occurs here

    println!("{}", r);
                   // - borrow later used here
}
```

```rust
fn main() {
    let mut a = 1;
    let r = &a;             // 或&mut a

    println!("{}", r);      // 后续不再使用

    a = 50;                 // OK!
}
```

&nbsp;

同一时刻，不能同时有可变和不可变借用。

```rust
fn main() {
    let mut x = 1;
    let r = &mut x;

    println!("{}{}", x, r);
                     // ^  - mutable borrow later used here
                     // |
                     // immutable borrow occurs here
}
```

```rust
fn main() {
    let mut x = 5;
    let r = &mut x;

    *r += 1;

    println!("{}", x);
                   // ^ immutable borrow occurs here 
    println!("{}", *r);
                   // -- mutable borrow later used here
}
```

```rust
fn main() {
    let mut x = 5;
    
    {
        let r = &mut x;
        *r += 1;
        println!("{}", *r);
    }                           // 可变借用结束

    println!("{}", x);
}
```

&nbsp;

同一时刻，只能有一个可写。

```rust
fn main() {
    let mut x = 1;
    let r1 = &mut x;
             // ------ first mutable borrow occurs here
    let r2 = &mut x;
             // ^^^^^^ second mutable borrow occurs here

    println!("{}{}", r1, r2);
            // -- first borrow later used here
}
```

值可变和引用自身可变的区别。

> 和指针一样，引用自身也是一个对象，有存储空间。
> 
> 调试器或反汇编，看到引用与指针一样.

&nbsp;

```rust
fn main() {
    let mut x = 100;
    let r = &mut x;             // 值可变，let r: &mut i32 = &mut x;
    *r += 1;
    
    assert_eq!(*r, 101);
}
```

```rust
fn main() {
    let x = 100;
    let y = 200;

    let mut r = &x;             // 引用自身可变。 let mut r: &i32 = &x;
    assert_eq!(*r, x);

    r = &y;                     // 引用绑定新对象
    assert_eq!(*r, y);
}
```
