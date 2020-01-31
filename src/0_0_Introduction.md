# Futures Explained in 200 Lines of Rust

This book aims to explain `Futures` in Rust using an example driven approach.

The goal is to get a better understanding of `Futures` by implementing a toy
`Reactor`, a very simple `Executor` and our own `Futures`. 

We'll start off a bit untraditionally. Instead of deferring some of the details about what's
special about futures in Rust we try to tackle that head on first. We'll be as brief as possible,
but as thorough as needed. I findt that implementing and understanding `Futures` is a lot easier
then. Actually, most questions will be answered up front. 

We'll end up with futures that can run an any executor like `tokio` and `async_str`.

In the end I've made some reader excercises you can do if you want to fix some
of the most glaring ommissions and shortcuts we took and create a slightly better
example yourself.

## What does this book give you that isn't covered elsewhere?

That's a valid question. There are many good resources and examples already. First
of all, this book will point you to some background information that I have found
very valuable, especially `Generators` and stackless coroutines.

I find that many discussions arise, not because `Futures` is a hard concept to
grasp, but that concurrent programming is a hard concept in general.

Secondly, I've always found small runnable examples very exiting to learn from. It's
all code that you can download, play with and learn from.

## What we'll do and not

**We'll:**

- Implement our own `Futures` and get to know the `Reactor/Executor` pattern
- Implement our own waker and learn why it's a bit foreign compared to other types
- Talk a bit about runtime complexity and what to keep in mind when writing async Rust.
- Make sure all examples can be run on the playground
- Not rely on any helpers or libraries, but try to face the complexity and learn

**We'll not:**

- Talk about how futures are implemented in Rust the language, the state machine and so on
- Explain how the different runtimes differ, however, you'll hopefully be a bit
better off if you read this before you go research them
- Explain concurrent programming, but I will supply sources

I do want to explore Rusts internal implementation but that will be for a later
book.

## Credits and thanks

I'll like to take the chance of thanking the people behind `mio`, `tokio`, 
`async_std`, `Futures`, `libc`, `crossbeam` and many other libraries which so
much is built upon. Reading and exploring some of this code is nothing less than
impressive.

## Why is `Futures` in Rust hard to understand

Well, I think it has to do with several things:

1. Futures has a very interesting implementation, compiling down to a state
machine using generators to suspend and resume execution. In a language such as
Rust this is pretty hard to do ergonomically and safely. You are exposed to some
if this complexity when working with futures and want to understand them, not
only learn how to use them.

2. Rust doesn't provide a runtime. That means you'll actually have to choose one
yourself and actually know what a `Reactor` and an `Executor` is. While not
too difficult, you need to make more choices than you need in GO and other
languages designed with a concurrent programming in mind and ships with a
runtime.

3. Futures exist in two versions, Futures 1.0 and Futures 3.0. Futures 1.0 was
known to have some issues regarding ergonomics. Turns out that modelling 
async coding after `Promises` in JavaScript can turn in to extremely long errors 
and type signatures with a type system as Rust.

Futures 3.0 are not compatible with Futures 1.0 without performing some work.

4. Async await syntax was recently stabilized

what we'll
really do is to stub out a `Reactor`, and `Executor` and implement


