# Some background information

> **Relevant for:**
>
> - High level introduction to concurrency in Rust
> - Knowing what Rust provides and not when working with async code
> - Understanding why we need runtimes 
> - Getting pointers to further reading on concurrency in general

Before we start implementing our `Futures` , we'll go through some background
information that will help demystify some of the concepts we encounter.

Actually, after going through these concepts, implementing futures will seem
pretty simple. I promise.

## Async in Rust

Let's get some of the common roadblocks out of the way first.

Languages like C#, JavaScript, Java, GO and many others comes with a runtime
for handling concurrency. So if you come from one of those languages this will
seem a bit strange to you.

Rust is different from these languages in the sense that Rust doesn't come with
a runtime for handling concurrency, so you need to use a library which provide
this for you.

In other words you'll have to make an active choice about which runtime to use
which will of course seem foreign if the environment you come from provides one
which "everybody" uses.

### What Rust's standard library takes care of

1. A common interface representing an operation which will be completed in the
future through the `Future` trait.
2. An ergonomic way of creating tasks which can be suspended and resumed through
the `async` and `await` keywords.
3. A defined interface wake up a suspended task through the `Waker` trait.

That's really what Rusts standard library does. As you see there is no definition
of non-blocking I/O, how these tasks are created or how they're run.

### What you need to find elsewhere

A runtime, often just referred to as an `Executor`.

There are mainly two such runtimes in wide use in the community today 
[async_std][async_std] and [tokio][tokio].

Executors, accepts one or more asynchronous tasks (`Futures`) and takes
care of actually running the code we write, suspend the tasks when they're
waiting for I/O and resume them when they can make progress.

>Now, you might stumble upon articles/comments which mentions both an `Executor`
and an `Reactor` (also referred to as a `Driver`) as if they're well defined
concepts you need to know about. This is not true. In practice today you'll only
interface with the runtime, which provides leaf futures which actually wait for
some I/O operation, and the executor where  

## Futures

Now, when we talk about futures I find it useful to make a distinction between
futures created by async functions `async fn() { ... }` and async blocks
`async { ... }` and **leaf** futures.

Runtimes create leaf `Futures`, and provides things like non-blocking sockets,
an event queue and so on.



In theory, we could choose one `Reactor` and one `Executor` that have nothing
to do with each other besides that one creates leaf `Futures` and the other one
runs them, but in reality today you'll most often get both in a `Runtime`.



Quite a bit of complexity attributed to `Futures` are actually complexity rooted
in runtimes. Creating an efficient runtime is hard. 

Learning how to use one correctly can require quite a bit of effort as well, but you'll see that there are several similarities between these kind of runtimes so
learning one makes learning the next much easier.

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

While they're not directly compatible, there is a tool that let's you relatively
easily convert a `Future 1.0` to a `Future 3.0` and vice a versa. You can find
all you need in the [`futures-rs`][futures_rs] crate and all 
[information you need here][compat_info].

## First things first

If you find the concepts of concurrency and async programming confusing in
general, I know where you're coming from and I have written some resources to 
try to give a high level overview that will make it easier to learn Rusts 
`Futures` afterwards:

* [Async Basics - The difference between concurrency and parallelism](https://cfsamson.github.io/book-exploring-async-basics/1_concurrent_vs_parallel.html)
* [Async Basics - Async history](https://cfsamson.github.io/book-exploring-async-basics/2_async_history.html)
* [Async Basics - Strategies for handling I/O](https://cfsamson.github.io/book-exploring-async-basics/5_strategies_for_handling_io.html)
* [Async Basics - Epoll, Kqueue and IOCP](https://cfsamson.github.io/book-exploring-async-basics/6_epoll_kqueue_iocp.html)

Learning these concepts by studying futures is making it much harder than
it needs to be, so go on and read these chapters if you feel a bit unsure. 

I'll be right here when you're back.

However, if you feel that you have the basics covered, then let's get moving!

[async_std]: https://github.com/async-rs/async-std
[tokio]: https://github.com/tokio-rs/tokio
[compat_info]: https://rust-lang.github.io/futures-rs/blog/2019/04/18/compatibility-layer.html
[futures_rs]: https://github.com/rust-lang/futures-rs