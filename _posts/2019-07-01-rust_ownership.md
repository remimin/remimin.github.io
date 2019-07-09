---
layout:     post
title:      "Rust学习之理解Ownership"
date:       2019-07-01
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - Rust
---
# Rust学习之理解Ownership

[toc]

> Rust语言的ownership是rust语言的核心，rust语言之所以被称之为安全的面向系统级别的编程语言
> 正是由此特性决定的。



## Rust ownership rule
首先我们先了解rust ownership的规则，这是很关键的，在了解Rust规则的基础上，再结合后续内容去理解
Rust语言的安全性。

1. Each value in Rust has a variable that's called its owner. #在Rust中，每一个值都有一个owner的变量
2. There can only be one owner at a time. #每次只能存在一个owner
3. When the owner goes out of scope, the value will be dropped. # 当owner超出作用域时，value也会删除

## Rust内存管理
Rust内存管理不同于c/c++语言，需要显示的alloca/free，在c/c++中，无法防止对无效内存的访问，当出现错误时开发者只能分析代码去找出越界指针。Rust也不像Java，没有GC机制不会定期的检测无用数据去释放内存空间。
rust是由数据的owner根据scope（作用域）来自动的释放（drop）。

### 栈和堆（Stack & heap）
栈同其他编程语言一样，先进后出的结构，在Rust中栈只能存储size固定的变量。而堆则用于存储空间需求不明确的变量。例如
```
let s = "hello word" # 存储在stack
let mut s = String::from("hello word") #存储在堆
```

### 变量作用域（Variable Scope）
在Rust中，作用域是一个关键的概念，RUST是没有GC操作的，变量申请空间的回收机制则是通过作用域完成的，不论是数据存在堆还是栈。
当变量存储空间在**栈**上时，作用域如下， 当变量`s`在代码块之外时，内存空间就会被自动回收。
```
{
    // s is not valid here, it’s not yet declared
    let s = "hello"; // s is valid from this point forward
    // do stuff with s 
} // this scope is now over, and s is no longer valid
```
当变量存储在**堆**上时，当变量`s`超出作用时，rust自动调用`drop`函数回收变量`s`在堆上的占用空间

```
fn main() {
    {
        let s = String::from("hello"); // s is valid from this point forward

        // do stuff with s
    }        // this scope is now over, and s is no
             // longer valid
}
```
### 变量赋值操作

#### 栈赋值
对于**栈**上数据的赋值操作，见代码块，对应内存的动作是，首先申请一个固定空间在栈上，内存存储数据5，并将变量x绑定到值5；然后做拷贝x对应的值5到栈中新的内存空间，并绑定到变量y。
```
fn main() {
let x = 5;
let y = x;
}
```

#### 堆赋值
**堆**数据的赋值操作， 实际上做的是指针拷贝，即不会拷贝String变量数据，s2仅拷贝了s1的指针数据，即两个变量指向堆的同一块内存，类似与python中的浅拷贝（shallow copy）。必须注意的是，前面我们在讨论作用域时说过，当堆上的变量超出作用域时，自动调用`drop` 去释放空间，那么如果做赋值操作两个变量s1和s2都指向同一个内存空间，rust会调用两次`drop`去释放空间吗？答案当然是否定的，请参考ownership的规则的第二条，值只会拥有**唯一**的owner。那么就以为s1赋值给s2时，值得owner就成为了s2，s1不再具有访问堆上数据得权限。由于s1不再具有访问权限，rust中将这种赋值拷贝称为**move**而不是浅拷贝。
```
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
    println!("{}, world!", s1); #Get Error  use of moved value: `s1`
}
```
**堆**上数据需要深拷贝（deep copy）时，则需要显示调用**clone**方法，堆数据的拷贝是代价较高的操作，因为rust中堆都是预先分配很大空间，**clone**意味着作了大的内存拷贝。
```
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();
    println!("s1 = {}, s2 = {}", s1, s2);
}
```
### 栈数据copy trait

上述可以了解到**堆**数据是必须通过**clone**这个代价较高的操作完成deep copy。而栈上数据并不需要，这是因为栈上存储的数据必须是编译期间确定内存空间的数据，因为数据的拷贝操作是固定空间代价小。Rust中有个**copy**trait，对于自定的类型数据需要如果实现copy trait，那么旧的变量在赋值操作后依然是可用的。**copy**和**drop**是两个冲突的trait，即当类型中有drop trait，那么**copy** trait则是不允许的，编译就会报错。
Rust中允许**copy**的类型
- All the integer types, such as u32.
- The Boolean type, bool, with values true and false.
- All the floating point types, such as f64.
- The character type, char.
- Tuples, if they only contain types that are also Copy. For example, (i32, i32) is Copy, but (i32, String) is not.

## 函数与ownership

### 参数传递
函数调用的参数传递，与赋值操作类似，就是变量的*move*或者*copy*。
```
fn main() {
    let s = String::from("hello");  // s comes into scope

    takes_ownership(s);   // s's value moves into the function...
                                    // ... and so is no longer valid here

    let x = 5;                      // x comes into scope

    makes_copy(x);                  // x would move into the function,
                                    // but i32 is Copy, so it’s okay to still
                                    // use x afterward

} // Here, x goes out of scope, then s. But because s's value was moved, nothing
  // special happens.

fn takes_ownership(some_string: String) { // some_string comes into scope
    println!("{}", some_string);
} // Here, some_string goes out of scope and `drop` is called. The backing
  // memory is freed.

fn makes_copy(some_integer: i32) { // some_integer comes into scope
    println!("{}", some_integer);
} // Here, some_integer goes out of scope. Nothing special happens.

```

### 函数返回值
函数返回值也会引起值的ownership的转移, 如下：

```
fn main() {
    let s1 = gives_ownership();         // gives_ownership moves its return
                                        // value into s1

    let s2 = String::from("hello");     // s2 comes into scope

    let s3 = takes_and_gives_back(s2);  // s2 is moved into
                                        // takes_and_gives_back, which also
                                        // moves its return value into s3
} // Here, s3 goes out of scope and is dropped. s2 goes out of scope but was
  // moved, so nothing happens. s1 goes out of scope and is dropped.

fn gives_ownership() -> String {             // gives_ownership will move its
                                             // return value into the function
                                             // that calls it

    let some_string = String::from("hello"); // some_string comes into scope

    some_string                              // some_string is returned and
                                             // moves out to the calling
                                             // function
}

// takes_and_gives_back will take a String and return one
fn takes_and_gives_back(a_string: String) -> String { // a_string comes into
                                                      // scope

    a_string  // a_string is returned and moves out to the calling function
}

```

## 引用
上面介绍了当数据在**堆**上时，赋值操作，函数调用，函数返回都会引起值ownership的变化，如果我们想在函数调用后，继续使用旧的变量，就要用到rust的引用（reference）机制。函数`calculate_length`的参数改变为参数的引用，那么就不会引入move操作，即ownership的切换。变量`s`时变量`s1`的引用而不是数据的owner。Rust管引用操作成为借用（borrowing），引用期间值是不可变的，如change函数就会引起编译错误。

```
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
    change（&s1)
}

fn calculate_length(s: &String) -> usize {
    s.len()
} //Here, s goes out of scope. But because it does not have ownership of what
// it refers to, nothing happens.


fn change(some_string: &String) { 
    some_string.push_str(", world");  //compile error
 }

```
![](/img/2019-07-01_rust_ownership/reference.png)

### 可变引用 (mutable reference)
上述的例子中，String变量`s`是不可变的，引用s1也是不可变的，因此在调用函数中`s1`的值是无法变更的，例如，使用String引用去更改s的内容，在编译时会报错，显示变量时不可变的。

```
fn main() {
    let s = String::from("hello");

    change(&s);
}

fn change(some_string: &String) {
    some_string.push_str(", world"); //error: cannot borrow immutable borrowed content `*some_string` as mutable
}

```
那么如果想要使用可变的引用，在rust中要怎么做呢，要分为两部分

- 变量本身应该是可变的变量
- 变量的引用是可变变量的引用
那么上述例子，改为如下，即可正常运行
```
fn main() {
    let mut s = String::from("hello"); //可变变量 s

    change(&mut s); //可变引用
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}

```
在Rust中，使用mut reference是由严格限制的：

- 可变引用只能存在一个
- 不可变应用与可变用不可同时存在

怎么理解呢，mut ferenece可以看作变量值的读写锁，immutable reference可以看作变量值的读锁（当然事实上并不是变量的锁）immutable reference是可以同时存在多个，没有问题，但是mutable reference，**只能存在一个且不能与immutable reference同时存在**
Rust做严格的引用限制的原因，目的就是**避免竞争**，不需要使用锁机制，从语言**编译**阶段检测是否存在这种竞争，如果存在则会编译报错，从而保证语言代码本身的变量是不能存在竞争。

那么看下面两段代码，第一段代码，编译会出错，原因是r1, r2是两个不可变引用，而r3是可变引用，且三个变量存在相同的scope中；
而第二段代码，在编译正常，那么是r1， r2的使用在r3引入之前，即r1， r2和r3的scope是没有重叠
```
let mut s = String::from("hello");

let r1 = &s; // no problem
let r2 = &s; // no problem
let r3 = &mut s; // BIG PROBLEM

println!("{}, {}, and {}", r1, r2, r3);

```

```
let mut s = String::from("hello");

let r1 = &s; // no problem
let r2 = &s; // no problem
println!("{} and {}", r1, r2);
// r1 and r2 are no longer used after this point

let r3 = &mut s; // no problem
println!("{}", r3);
```

### 切片类型（ Slice Type）
之前看到了Rust的引用，是不会具有ownership的类型，在Rust中还有一种数据类型也不具有ownership就是`slice`切片。slice就是对一个集和中连续元素的**引用**。
以String为例，slice是对string 的部分引用，而不是整体。String切片类型写成`&str`。 

那么回顾一下Rust的字符串常量`string literals` (不是String类型)，
`let s = "hello word"`， 字符串常量其实就是切片类型`&str`，切片指针指向是一段固定的内存区域，且不可变。

下面这段代码有助理解`&str`

```
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
fn main() {
    let my_string = String::from("hello world");

    // first_word works on slices of `String`s
    let word = first_word(&my_string[..]);

    let my_string_literal = "hello world";

    // first_word works on slices of string literals
    let word = first_word(&my_string_literal[..]);

    // Because string literals *are* string slices already,
    // this works too, without the slice syntax!
    let word = first_word(my_string_literal);
}
```
上述内容仅仅是以String作为切片的举例，Rust中的切片支持其他的集合类型的切片，例如整型数组
```
let a = [1, 2, 3, 4, 5]; 
let slice = &a[1..3];
```
## 总结
Rust中的ownership, borrowing, slices这些概念是Rust在编译期间保证内存安全的关键因素。因此Rust编程是需要使用者
关注内存使用的，数据的owner超出作用域时，由owner自回收的。超出作用域范围再使用数据Rust编译就会报错。

## 参考链接
[1] [Rust编程指南](https://doc.rust-lang.org/book/title-page.html)