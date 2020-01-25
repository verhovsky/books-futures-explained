# Some background information

Before we start implementing our `Futures`, we'll go through some background
information that will help demystify some of the concepts we encounter.

## Concurrency in general

If you find the concepts of concurrency and async programming confusing in
general, I know where you're coming from and I have written some resources to 
try to give a high level overview that will make it easier to learn Rusts 
`Futures` afterwards:

[Async Basics - The difference between concurrency and parallelism](https://cfsamson.github.io/book-exploring-async-basics/1_concurrent_vs_parallel.html)
[Async Basics - Async history](https://cfsamson.github.io/book-exploring-async-basics/2_async_history.html)
[Async Basics - Strategies for handling I/O](https://cfsamson.github.io/book-exploring-async-basics/5_strategies_for_handling_io.html)
[Async Basics - Epoll, Kqueue and IOCP](https://cfsamson.github.io/book-exploring-async-basics/6_epoll_kqueue_iocp.html)

## Trait objects and dynamic dispatch

The single most confusing topic we encounter when implementing our own `Futures`
is how we implement a `Waker`. Creating a `Waker` involves creating a `vtable`
which allows using dynamic dispatch to call methods on a _type erased_ trait 
object we construct our selves.

If you want to know more about dynamic dispatch in Rust I can recommend this article:

https://alschwalm.com/blog/static/2017/03/07/exploring-dynamic-dispatch-in-rust/


Let's explain this a bit more in detail.

## Fat pointers in Rust

Let's take a look at the size of some different pointer types in Rust. If we
run the following code:

```rust
# use std::mem::size_of;
trait SomeTrait { }

fn main() {
    println!("Size of Box<i32>: {}", size_of::<Box<i32>>());
    println!("Size of &i32: {}", size_of::<&i32>());
    println!("Size of &Box<i32>: {}", size_of::<&Box<i32>>());
    println!("Size of Box<Trait>: {}", size_of::<Box<SomeTrait>>());
    println!("Size of &dyn Trait: {}", size_of::<&dyn SomeTrait>());
    println!("Size of Rc<Trait>: {}", size_of::<Rc<SomeTrait>>());
    println!("Size of &[i32]: {}", size_of::<&[i32]>());
    println!("Size of &[&dyn Trait]: {}", size_of::<&[&dyn SomeTrait]>());
    println!("Size of [i32; 10]: {}", size_of::<[i32; 10]>());
    println!("Size of [&dyn Trait; 10]: {}", size_of::<[&dyn SomeTrait; 10]>());
}
```