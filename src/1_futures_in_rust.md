# Futures in Rust

> **Overview:**
>
> - High level introduction to concurrency in Rust
> - Knowing what Rust provides and not when working with async code
> - Understanding why we need a runtime-library in Rust
> - Getting pointers to further reading on concurrency in general

## Futures

So what is a future?

A future is a representation of some operation which will complete in the
future.

Async in Rust uses a `Poll` based approach, in which an asynchronous task will
have three phases.

1. **The Poll phase.** A Future is polled which result in the task progressing until
a point where it can no longer make progress. We often refer to the part of the
runtime which polls a Future as an executor.
2. **The Wait phase.** An event source, most often referred to as a reactor,
registers that a Future is waiting for an event to happen and makes sure that it
will wake the Future when that event is ready.
3. **The Wake phase.** The event happens and the Future is woken up. It's now up
to the executor which polled the Future in step 1 to schedule the future to be
polled again and make further progress until it completes or reaches a new point
where it can't make further progress and the cycle repeats.

Now, when we talk about futures I find it useful to make a distinction between
**non-leaf** futures and **leaf** futures early on because in practice they're
pretty different from one another.

### Leaf futures

Runtimes create _leaf futures_ which represents a resource like a socket.

```rust, ignore, noplaypen
// stream is a **leaf-future**
let mut stream = tokio::net::TcpStream::connect("127.0.0.1:3000");
```

Operations on these resources, like a `Read` on a socket, will be non-blocking
and return a future which we call a leaf future since it's the future which
we're actually waiting on.

It's unlikely that you'll implement a leaf future yourself unless you're writing
a runtime, but we'll go through how they're constructed in this book as well.

It's also unlikely that you'll pass a leaf-future to a runtime and run it to
completion alone as you'll understand by reading the next paragraph.

### Non-leaf-futures

Non-leaf-futures is the kind of futures we as _users_ of a runtime writes
ourselves using the `async` keyword to create a **task** which can be run on the
executor.

The bulk of an async program will consist of non-leaf-futures, which are a kind
of pause-able computation. This is an important distinction since these futures represents a _set of operations_. Often, such a task will `await` a leaf future
as one of many operations to complete the task.

```rust, ignore, noplaypen
// Non-leaf-future
let non_leaf = async {
    let mut stream = TcpStream::connect("127.0.0.1:3000").await.unwrap();// <- yield
    println!("connected!");
    let result = stream.write(b"hello world\n").await; // <- yield
    println!("message sent!");
    ...
};
```

The key to these tasks is that they're able to yield control to the runtime's
scheduler and then resume execution again where it left off at a later point.

In contrast to leaf futures, these kind of futures does not themselves represent
an I/O resource. When we poll these futures we either run some code or we yield
to the scheduler while waiting for some resource to signal us that it's ready so
we can resume where we left off.

## Runtimes

Languages like C#, JavaScript, Java, GO and many others comes with a runtime
for handling concurrency. So if you come from one of those languages this will
seem a bit strange to you.

Rust is different from these languages in the sense that Rust doesn't come with
a runtime for handling concurrency, so you need to use a library which provide
this for you.

Quite a bit of complexity attributed to `Futures` are actually complexity rooted
in runtimes. Creating an efficient runtime is hard.

Learning how to use one correctly requires quite a bit of effort as well, but
you'll see that there are several similarities between these kind of runtimes so
learning one makes learning the next much easier.

The difference between Rust and other languages is that you have to make an
active choice when it comes to picking a runtime. Most often, in other languages
you'll just use the one provided for you.

**An async runtime can be divided into two parts:**

1. The Executor
2. The Reactor

When Rusts Futures were designed there was a desire to separate the job of
notifying a `Future` that it can do more work, and actually doing the work
on the `Future`.

You can think of the former as the reactor's job, and the latter as the
executors job. These two parts of a runtime interacts using the `Waker` type.

The two most popular runtimes for `Futures` as of writing this is:

- [async-std](https://github.com/async-rs/async-std)
- [Tokio](https://github.com/tokio-rs/tokio)

### What Rust's standard library takes care of

1. A common interface representing an operation which will be completed in the
future through the `Future` trait.
2. An ergonomic way of creating tasks which can be suspended and resumed through
the `async` and `await` keywords.
3. A defined interface wake up a suspended task through the `Waker` type.

That's really what Rusts standard library does. As you see there is no definition
of non-blocking I/O, how these tasks are created or how they're run.

## Bonus section

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