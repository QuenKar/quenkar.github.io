---
title: "memory ordering"
date: 2023-09-07T17:33:00+08:00
draft: false
categories: ["rust"]
---

最近在看*Rust Atomics and Locks*这本书，里面有提到memory ordering的相关问题，这里做一个总结。
常见的memory ordering有以下几种：
Relaxed, Release/Acquire, SeqCst，从左到右依次增加了约束。

## Happens-Before 关系
内存模型根据*Happens-Before*关系来定义操作发生的顺序。这意味着，作为一个抽象模型，它不谈论机器指令、缓存、缓冲区、定时、指令重序、编译器优化等，只定义一件事保证在另一件事之前发生的情况，而其他所有内容的顺序则是未定义的。

最基本的一个 *Happens-Before*关系就是在同一线程中，语句的顺序执行顺序。如果一个thread执行
`f();g();`，那么程序就是先执行`f()`，在执行`g()`。

下面来看看两个线程中的情况。
```rust
static X: AtomicI32 = AtomicI32::new(0);

fn main() {
    X.store(1, Relaxed);
    let t = thread::spawn(f);
    X.store(2, Relaxed);
    t.join().unwrap();
    X.store(3, Relaxed);
}

fn f() {
    let x = X.load(Relaxed);
    assert!(x == 1 || x == 2);
}
```
对于上面的这个语句块，我们可以得到以下的Happens-Before关系：

![v3tZP9n.png](https://i.imgur.com/v3tZP9n.png)

可以看出这个`assert!`是不会失败的，因为`X.load(Relaxed)`要么是1，要么是2。

## Ordering::Relaxed
针对单个atomic操作，没有任何约束，可以看作是没有memory ordering的操作。但实际上还有有一点约束的，唯一的顺序是，一旦在线程2中观察到来自线程1的变量的值，线程2就无法看到来自线程1的该变量的“更早”值。

```rust
fn test_relaxed() {
    println!("Relaxed Ordering");
    // create a new atomic u32
    let a_num = Arc::new(AtomicU32::new(0));
    let a_num_ref = Arc::clone(&a_num);

    // create 2 threads
    // use relaxed ordering to increment the number
    let t1 = std::thread::spawn(move || {
        for i in 0..1000000 {
            a_num.store(i, std::sync::atomic::Ordering::Relaxed);
        }
    });

    // use relaxed ordering to print the number, this will be monotonous increment
    let t2 = std::thread::spawn(move || {
        for _ in 0..100 {
            println!("{:}", a_num_ref.load(std::sync::atomic::Ordering::Relaxed))
        }
    });

    t1.join().unwrap();
    t2.join().unwrap();
}
```
result:
```css
Relaxed Ordering
82
2548
2649
2722
2791
2857
2924
...
```


## Ordering::Release/Ordering::Acquire

Release和Acquire是成对出现的，Release用于写操作，Acquire用于读操作。Release和Acquire的约束是，Release之前的所有操作都不能被Acquire之后的操作所观察到。这里的约束是针对单个atomic操作的，如果有多个atomic操作，那么这些操作之间的顺序是不确定的。

```rust
fn test_release_acquire() {
    println!("Release-Acquire Ordering");
    let data = Arc::new(AtomicBool::new(false));
    let data_clone = Arc::clone(&data);

    let t1 = std::thread::spawn(move || {
        // use release ordering to store the data
        data.store(true, Ordering::Release);
    });

    let t2 = std::thread::spawn(move || {
        // use acquire ordering to load the data
        let value = data_clone.load(Ordering::Acquire);
        println!("Value: {}", value);
    });

    t1.join().unwrap();
    t2.join().unwrap();
}
```
result:
```css
Release-Acquire Ordering
Value: true
```
这里需要特别注意的是：Acquire和Release的约束是针对单个atomic操作的，如果有多个atomic操作，那么这些操作之间的顺序是不确定的。我们可以看下面这个例子：

假设x和y都是原子变量，x和y的初始值都是0，线程1和线程2分别对x和y进行store操作，线程3和线程4分别对x和y进行load操作，那么线程3和线程4观察到的x和y的值可能是(0, 0), (0, 10), (20, 0), (20, 10)。
```css
init: x = 0 and y = 0

 -Thread 1-
 y.store (20, memory_order_release);

 -Thread 2-
 x.store (10, memory_order_release);

 -Thread 3-
 assert (y.load (memory_order_acquire) == 20 && x.load (memory_order_acquire) == 0)

 -Thread 4-
 assert (y.load (memory_order_acquire) == 0 && x.load (memory_order_acquire) == 10)
```
## Ordering::SeqCst

*It provides the same restrictions and limitation to moving loads around that sequential programmers are inherently familiar with, except it applies across threads.*

SeqCst是最强的约束，它可以保证所有线程观察到的操作顺序都是一致的。



```rust
fn test_seqcst() {
    println!("Seq-Cst Ordering");
    // create a new atomic u32
    let x = Arc::new(AtomicU32::new(0));
    let y = Arc::new(AtomicU32::new(0));
    let yy = Arc::clone(&y);
    let xx = Arc::clone(&x);

    let t1 = std::thread::spawn(move || {
        println!("t1: change x to 10, then y to 20");
        y.store(20, Ordering::SeqCst);
        x.store(10, Ordering::SeqCst);
    });

    let x = Arc::clone(&xx);
    let y = Arc::clone(&yy);

    let t2 = std::thread::spawn(move || {
        if x.load(Ordering::SeqCst) == 10 {
            println!("t2: x is 10 and y is 20");
            assert_eq!(y.load(Ordering::SeqCst), 20);
            println!("t2: change y to 5");
            y.store(5, Ordering::SeqCst);
        }
    });

    let t3 = std::thread::spawn(move || {
        if yy.load(Ordering::SeqCst) == 5 {
            println!("t3: x is 10 and y is 5");
            assert_eq!(xx.load(Ordering::SeqCst), 10);
        }
    });

    t1.join().unwrap();
    t2.join().unwrap();
    t3.join().unwrap();
}
```
result:
```css
Seq-Cst Ordering
init: x is 0 and y is 0
t1: change x to 10, then y to 20
t2: x is 10 and y is 20
t2: change y to 5
t3: x is 10 and y is 5
```

## Reference

https://gcc.gnu.org/wiki/Atomic/GCCMM/AtomicSync

https://marabos.nl/atomics/