# Reactor/Executor Pattern

> **Relevant for:**
>
> - Getting a high level overview of a common runtime model in Rust
> - Introducing these terms so we're on the same page when referring to them
> - Getting pointers on where to get more information about this pattern

If you don't know what this is, you should take a few minutes and read about
it. You will encounter the term `Reactor` and `Executor` a lot when working
with async code in Rust.

I have written a quick introduction explaining this pattern before which you
can take a look at here:


[![homepage][1]][2]

<div  style="text-align:center">
<a href="https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/appendix-1/reactor-executor-pattern">Epoll, Kqueue and IOCP Explained - The Reactor-Executor Pattern</a>
</div>

I'll re-iterate the most important parts here.

**This pattern consists of at least 2 parts:**

1. **A reactor**
    - handles some kind of event queue
    - has the responsibility of respoonding to events
2. **An executor**
    - Often has a scheduler
    - Holds a set of suspended tasks, and has the responsibility of resuming
    them when an event has occurred
3. **The concept of a task**
    - A set of operations that can be stopped half way and resumed later on

This kind of pattern common outside of Rust as well, but it's especially popular in Rust due to how well it alignes with the API provided by Rusts standard library. This model separates concerns between handling and scheduling tasks, and queing and responding to I/O events.

## The Reactor

Since concurrency mostly makes sense when interacting with the outside world (or
at least some peripheral), we need something to actually abstract over this
interaction in an asynchronous way. 

This is the `Reactors` job. Most often you'll
see reactors in rust use a library called [Mio][mio], which provides non 
blocking APIs and event notification for several platforms.

The reactor will typically give you something like a `TcpStream` (or any other resource) which you'll use to create an I/O request. What you get in return
is a `Future`. 

We can call this kind of `Future` a "leaf Future`, since it's some operation 
we'll actually wait on and that we can chain operations on which are performed
once the leaf future is ready. 

## The Task

In Rust we call an interruptible task a `Future`. Futures has a well defined interface, which means they can be used across the entire ecosystem. We can chain
these `Futures` so that once a "leaf future" is ready we'll perform a set of
operations. 

These operations can spawn new leaf futures themselves.

## The executor

The executors task is to take one or more futures and run them to completion.

The first thing an `executor` does when it get's a `Future` is polling it.

**When polled one of three things can happen:**

- The future returns `Ready` and we schedule whatever chained operations to run
- The future hasn't been polled before so we pass it a `Waker` and suspend it
- The futures has been polled before but is not ready and returns `Pending`

Rust provides a way for the Reactor and Executor to communicate through the `Waker`. The reactor stores this `Waker` and calls `Waker::wake()` on it once
a `Future` has resolved and should be polled again.

We'll get to know these concepts better in the following chapters.

Providing these pieces let's Rust take care a lot of the ergonomic "friction"
programmers meet when faced with async code, and still not dictate any
preferred runtime to actually do the scheduling and I/O queues.


With that out of the way, let's move on to actually implement all this in our
example.

[1]: ./assets/reactorexecutor.png
[2]: https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/appendix-1/reactor-executor-pattern
[mio]: https://github.com/tokio-rs/mio