---
title: 用 Rust 写一个泛型并查集
date: 2022-10-23 15:58:39
tags:
    - Rust
    - 并查集
categories: Rust
keywords: 'Rust, 数据结构, 并查集'
description: rust 编程踩坑纪实
cover: cover.jpg
---


首先是最初的简单想法，按其他编程语言的一般思维，要写一个并查集，立刻就会写下如下的数据结构：
```rust
use std::{
    collections::HashMap,
    hash::Hash
};
pub struct DisjointSet<T> {
    parent: HashMap<T, T>,
    rank: HashMap<T, usize>
}
```
那么接下来就是实现并查集有关的操作，第一个操作自然是创建一个并查集，我们可以从一个 `Vec<T>` 来构建：
```rust
impl<T: Eq + Hash> From<Vec<T>> for DisjointSet<T> {
    fn from(ori: Vec<T>) -> Self {
        unimplemented!()
    }
}
```
写到这个时候已经愣住，意识到自己写错了，因为可以注意到现在的 `DisjointSet` 是完全拥有数据的所有权的，对集合里的同一个元素，`parent` 和 `rank` 就要求三份所有权，这样一来 `from` 的返回值就将要求我们把 `ori` 中的数据额外拷贝两份，这样做太傻了，我们也不可能只为拥有 `trait Clone` 的类型实现并查集。

那么问题也很显然，就是一个并查集并不该拥有这么多所有权。于是立刻写下了如下的代码：
```rust
use std::{
    collections::HashMap,
    hash::Hash,
    rc::Rc,
    cell::RefCell
};
pub struct DisjointSet<T> {
    parent: HashMap<Rc<RefCell<T>>, Rc<RefCell<T>>>,
    rank: HashMap<Rc<RefCell<T>>, usize>
}
```
搞得很暴力，看得出选手十分暴躁，`Rc<RefCell<T>>` 多所有权且可变，这不是无敌吗。
```rust
impl<T: Eq + Hash> From<Vec<T>> for DisjointSet<T> {
    fn from(ori: Vec<T>) -> Self {
        let mut parent = HashMap::new();
        let mut rank = HashMap::new();
        for e in ori.into_iter() {
            let r = Rc::new(RefCell::new(e));
            parent.insert(r.clone(), r.clone());
            rank.insert(r.clone(), 1);
        }
        DisjointSet { parent, rank }
    }
}
```

然而

```shell
error[E0277]: the trait bound `RefCell<T>: Hash` is not satisfied
   --> src\blog\mod.rs:19:25
    |
19  |             rank.insert(r.clone(), 1);
    |                  ------ ^^^^^^^^^ the trait `Hash` is not implemented for `RefCell<T>`
    |                  |
    |                  required by a bound introduced by this call
    |
    = help: the trait `Hash` is implemented for `Rc<T>`
    = note: required because of the requirements on the impl of `Hash` for `Rc<RefCell<T>>`
```
。。。


`RefCell` 没实现 `Hash`，怎么回事呢？`RefCell` 是对 rust <em>内部可变性</em> 的抽象，使得不可变的变量发生变化的手段，那「能够变化的不可变量」的 Hash 值应该是什么呢？显然啥都不是，所以 rust 没有为 `RefCell<T>` 实现 `trait Hash` ，rust 不允许出现在一个指针的眼皮底下修改它指向的内存的情况发生，自然也不允许有人在一个确定的 Hash Code 眼皮底下修改产生它的 key 。

这中间又尝试了很多办法，思考了很多东西，但是没那么重要所以先不写了。

最后思考来的结果是：并查集是一种用来体现集合里元素之间的关系的数据结构，它没有必要拥有任何数据的所有权，这点有点像容器的迭代器，迭代器体现出来容器的顺序关系，而并查集则体现出了子集的关系。

那么没有任何所有权的数据结构，就应该写成这样：
```rust
use std::{
    collections::HashMap,
    hash::Hash
};

#[derive(Debug)]
pub struct DisjointSet<'a, T> {
    parent: HashMap<&'a T, &'a T>,
    rank: HashMap<&'a T, usize>
}
```
`parent` 只保存数据的引用，体现键与值的关系，而不拥有他们。

那么对于所有集合类的容器，都可以构建并查集，体现元素的关系，这是一种特征，就类似于 `Iterator` 。
```rust
pub trait ToDisjointSet<T> {
    fn disjoint_set(&self) -> DisjointSet<T>;
}
```

这里就可以为类型的容器实现这一 `trait`，以 `Vec<T>` 为例
```rust
impl<T: Eq + Hash> ToDisjointSet<T> for Vec<T> {
    fn disjoint_set(&self) -> DisjointSet<T> {
        let mut parent = HashMap::new();
        let mut rank = HashMap::new();
        for e in self.iter() {
            parent.insert(e, e);
            rank.insert(e, 1);
        }
        DisjointSet { parent, rank }
    }
}
```
这里不知道是否可以写得更优雅一点，如果直接 `self.iter.map(...)` 的话，则会由于迭代器的惰性而不会执行传给 `map` 的闭包，所以就简单 for 循环了。

接下来需要实现并查集的 `find(x)` 操作，即找到值 `x` 在并查集中的代表元素，我们还可能需要修改这一代表元素，所以方法大概应该会长成这样
```rust
impl<'a, T: Eq + Hash> DisjointSet<'a, T> {
    pub fn find_mut(&mut self, val: &T) -> Option<&mut &T> {
        unimplemented!()
    }
}
```
但这样的签名显然是不能通过的，这其中明显牵扯到很多生命周期的问题。

首先 `impl<'a, T: Eq + Hash> DisjointSet<'a, T>` 标识了并查集中引用的所有数据的生命周期为 `'a` ，那么首先就对 `find_mut` 的返回值的生命周期提出了要求，其应当为「引用了一个生命周期为 `'a` 的 `T` 类型变量的引用的可变引用」，用人话来说就是 `&mut &'a T`。其次，参数的 `val` 实质上就是我们返回的 value 所对应的 key ，在数据结构中的 key 和 value 的生命周期是相等的，那么显然 `val` 的生命周期应该和返回的值一眼，即 `val` 的类型应当是 `&'a T` 。

那 `self` 如何呢？考虑这样的调用：
```rust
fn main() {
    let v: Vec<&str> = vec!["123", "213", "321"];
    let mut s = v.disjoint_set();


    match s.find_mut(&"123") {
        None => (),
        Some(rmrs) => { *rmrs = &"abc"; }
    };

    println!("{:?}", s);
}
```
调用 `s.find_mut(&"123")` 时，发生了对 s 的可变借用，那么这个借用应该持续多长时间？很明显 `find_mut` 执行结束后，这个借用也就没有必要继续存在了。所以签名上的 `self` 应该是没有任何生命周期参数的。

那么假如真的给签名的形参上写了 `&'a mut self` ，会发生什么呢？

在上述调用代码中，执行 `s.find_mut` 时，发生了对 `s` 的可变借用。如果这个可变借用的生命周期是 `'a` 则这个可变借用将存活得与数组 `v` 一样长，即一直到达 `main` 的结束。那么在之后调用 `println!("{:?}", s);` 时，要求传入一个 `s` 的不可变引用，而这时程序还未到达 `main` 的结束，在 `find_mut` 中发生的可变借用仍然存在，这就将导致程序中同时存在 `s` 的一个可变借用和一个不可变借用。

显然，这违背了 rust 的所有权规则，编译器将不会允许这样的代码通过。
```shell
error[E0502]: cannot borrow `s` as immutable because it is also borrowed as mutable
  --> src\main.rs:14:22
   |
9  |     match s.find_mut(&"123") {
   |           ------------------ mutable borrow occurs here
...
14 |     println!("{:?}", s);
   |                      ^
   |                      |
   |                      immutable borrow occurs here
   |                      mutable borrow later used here
   |
   = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)
```

明确了函数签名，下面就可以来实现这个方法，最简单的符合直觉的写法应当是如下
```rust
impl<'a, T: Eq + Hash> DisjointSet<'a, T> {
    pub fn find_mut(&mut self, val: &'a T) -> Option<&'a mut & T> {
        match self.parent.get_mut(&val) {
            None => None,
            Some(s) => if *s == val { Some(s) } else { self.find_mut(*s) }
        }
    }
}
```
然而，符合直觉的东西通常是不会被 rust 接受的

```shell
error[E0499]: cannot borrow `*self` as mutable more than once at a time
  --> src\disjoint_set\mod.rs:36:56
   |
28 | impl<'a, T: Eq + Hash> DisjointSet<'a, T> {
   |      -- lifetime `'a` defined here
...
34 |         match self.parent.get_mut(&val) {
   |               ------------------------- first mutable borrow occurs here
35 |             None => None,
36 |             Some(s) => if *s == val { Some(s) } else { self.find_mut(*s) }
   |                                       -------          ^^^^^^^^^^^^^^^^^ second mutable borrow occurs here
   |                                       |
   |                                       returning this value requires that `self.parent` is borrowed for `'a`

For more information about this error, try `rustc --explain E0499`.
error: could not compile `data_process` due to previous error
```
好，继续和编译器斗智斗勇。编译器的意思是指，当我们进行递归调用 `self.find_mut(*s)` 时，出现了对 `self` 的多个可变借用，违反了 rust 的所有权规则。编译器还告诉我们，第一个可变借用就发生在 `match` 之后的这次 `self.parent.get_mut(&val)` 上。可是为什么会这样的？ `get_mut` 对 `self` 的可变借用应当和我们的 `find_mut` 一样，在函数结束后就结束了其 lifetime ，上例中我们已经获取到了 `get_mut` 返回的结果，这个函数已经执行完毕，它对 `self` 的可变借用应该已经消失，为什么还会影响我们的递归调用呢？

我在 rust 官方的 issue 里找到了[这个问题](https://github.com/rust-lang/rust/issues/84361)，和我们现在遇到的问题是一样的，在此找到了根据 rust 编译器产生的中间码来解释这个问题的答案。这是因为在 `match` 之后调用的这次 `get_mut` 造成的对 `self` 的可变借用，仅会在当前函数的 `return` 之前才会被释放。

>The problem here is that the reference created by the first self.match() is only made dead in this function right before a return. In slightly abridged-MIR:

目前我对 rust 的了解程度尚不足以支持我去深入探究这种现象的原因和编译器这样设计的考量，但这一解答足以让我们解决代码不能编译的问题。我们可以换用这样的方式实现 `find_mut`
```rust
impl<'a, T: Eq + Hash> DisjointSet<'a, T> {
    pub fn find_mut(&mut self, val: &'a T) -> Option<&'a mut &T> {
        if self.parent.get_mut(&val).is_none() {
            return None;
        }
        return if self.parent.get_mut(&val).unwrap() == &val {
            Some(self.parent.get_mut(&val).unwrap())
        } else {
            self.find_mut(&val)
        };
    }
}
```
这段代码通过了编译。
```shell
warning: field `rank` is never read
 --> src\disjoint_set\mod.rs:6:5
  |
4 | pub struct DisjointSet<'a, T> {
  |            ----------- field in this struct
5 |     parent: HashMap<&'a T, &'a T>,
6 |     rank: HashMap<&'a T, usize>,
  |     ^^^^
  |
  = note: `#[warn(dead_code)]` on by default
  = note: `DisjointSet` has a derived impl for the trait `Debug`, but this is intentionally ignored during dead code analysis

warning: `data_process` (bin "data_process") generated 1 warning
    Finished dev [unoptimized + debuginfo] target(s) in 0.96s
```

真的很不容易TT

实际上，我在写这段代码时，犯的主要的错误是将返回的 `&mut &'a T` 写成了 `&'a mut &T`，这样写能否通过编译呢，如果可以，按照和之前的例子一样编写 `main` 函数，这样的 `main` 函数能通过编译吗，如果不能，是为什么？如果有人读到了这篇博客的话，就留给你思考一下吧！