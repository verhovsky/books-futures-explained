# Some background information

Before we start implementing our `Futures` , we'll go through some background
information that will help demystify some of the concepts we encounter.

Actually, after going through these concepts, implementing futures will seem
pretty simple. I promise.

## Async in Rust

Let's get some of the common roadblocks out of the way first.

Async in Rust is different from most other languages in the sense that Rust
has an extremely lightweight runtime.

In languages like C#, JavaScript, Java and GO, the runtime is already there. So
if you come from one of those languages this will seem a bit strange to you.

### What Rust's standard library takes care of

1. The definition of an interruptible task
2. An extremely efficient technique to start, suspend, resume and store tasks
which are executed concurrently. 
3. A defined way to wake up a suspended task

That's really what Rusts standard library does. As you see there is no definition
of non-blocking I/O, how these tasks are created or how they're run.


### What you need to find elsewhere

A runtime. Well, in Rust we normally divide the runtime into two parts:

- The Reactor
- The Executor

Reactors create leaf `Futures`, and provides things like non-blocking sockets,
an event queue and so on.

Executors, accepts one or more asynchronous tasks called `Futures` and takes
care of actually running the code we write, suspend the tasks when they're
waiting for I/O and resumes them.

In theory, we could choose one `Reactor` and one `Executor` that have nothing
to do with each other besides one creates leaf `Futures` and one runs them, but
in reality today you'll most often get both in a `Runtime`.

There are mainly two such runtimes today [async_std][async_std] and [tokio][tokio].

Quite a bit of complexity attributed to `Futures` are actually complexity rooted
in runtimes. Creating an efficient runtime is hard. Learning how to use one
correctly can be hard as well, but both are excellent and it's just like 
learning any new library.

The difference between Rust and other languages is that you have to make an
active choice when it comes to picking a runtime. Most often you'll just use
the one provided for you.

## Futures 1.0 and Futures 3.0

I'll not spend too much time on this, but it feels wrong to not mention that
there have been several iterations on how async should work in Rust.

`Futures 3.0` works with the relatively new `async/await` syntax in Rust and
it's what we'll learn.

Now, since this is rather recent, you can encounter creates that use `Futures 1.0`
still. This will get resolved in time, but unfortunately it's not always easy
to know in advance.

A good sign is that if you're required to use combinators like `and_then` then
you're using `Futures 1.0`.

While not directly compatible, there is a tool that let's you relatively easily
convert a `Future 1.0` to a `Future 3.0` and vice a verca. You can find all you
need in the [`futures-rs`][futures_rs] crate and all [information you need here][compat_info].

## First things first

If you find the concepts of concurrency and async programming confusing in
general, I know where you're coming from and I have written some resources to 
try to give a high level overview that will make it easier to learn Rusts 
`Futures` afterwards:

* [Async Basics - The difference between concurrency and parallelism](https://cfsamson.github.io/book-exploring-async-basics/1_concurrent_vs_parallel.html)
* [Async Basics - Async history](https://cfsamson.github.io/book-exploring-async-basics/2_async_history.html)
* [Async Basics - Strategies for handling I/O](https://cfsamson.github.io/book-exploring-async-basics/5_strategies_for_handling_io.html)
* [Async Basics - Epoll, Kqueue and IOCP](https://cfsamson.github.io/book-exploring-async-basics/6_epoll_kqueue_iocp.html)

Now learning these concepts by studying futures is making it much harder than
it needs to be, so go on and read these chapters. I'll be right here when
you're back. 

However, if you feel that you have the basics covered, then go right on. The concepts we need to
learn are:

1. Trait Objects and fat pointers
2. Generators/stackless coroutines
3. Pinning, what it is and why we need it

Let's get moving!

[async_std]: https://github.com/async-rs/async-std
[tokio]: https://github.com/tokio-rs/tokio
[compat_info]: https://rust-lang.github.io/futures-rs/blog/2019/04/18/compatibility-layer.html
[futures_rs]: https://github.com/rust-lang/futures-rs