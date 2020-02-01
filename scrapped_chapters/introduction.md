# Introduction

In this short book we'll explore Rust's `Futures`and implement a self contained example including a fake `Reactor`, a simple `Executor`and our own `Futures`. This books aims to give an introduction to `Futures` and the basics of how you can implement your own.

### Futures in Rust

In contrast to many other languages, Rust doesn't come with a large runtime. The reason for this is easy to understand when you look at some of Rusts goals:

* Provide zero cost abstractions \(a runtime is never zero cost\)
* Usable in embedded scenarios

Actually, at one point, Rust provided green threads for handling `async` programming, but they were dropped before Rust hit 1.0. The road after that has been a long one, but it has always revolved around the `Future`trait. 

`Futures` in Rust comes in several versions, and that can be a source of some confusion for new users.

#### Futures 1.0

This was the first iteration on how zero cost async programming could be implemented in Rust. Rusts 1.0 `Futures` is used using `combinators`. This means that we used methods on the `Futures` object themselves to chain operations on them.

An example of this could look like:

```rust
let future = SomeAction::new();
let fut_value = future.and_then(|object| {
    object.new_action().and_then(|secondaction| {
        Ok(secondaction.get_data())
    })
};

let value = executor.block_on(fut_value).unwrap();

println!("{}", value);
```

As you can see, these chains quickly become long and hard to work with. Callback-hell comes to mind. Though you could try un-nest them in a way similar to using `Promises` in JavaScript, the type signatures you have to work with quickly become so long and unwieldy \(think multiple lines of code just for the signature\) that the ergonomics discouraged it. The error messages could fill a whole screen.

There were other issues as well, but the lack of ergonomics was one of the major ones.

#### Futures 2.0

#### Futures 3.0

This is the current iteration over `Futures` and the one we'll use in our examples. This iteration solved a lot of the problems with 1.0, especially concerning ergonimics.

The `async/await` syntax was designed in tandem with `Futures 3.0` and provides a much more ergonomic way to work with `Futures`:

```rust
async fn asyncfunc() -> i32 {
    let object = SomeAction::new().await.unwrap();
    let secondaction = object.new_action().await;
    secondaction.get_data().await
}

let value = executor.block_on(asyncfunc()).unwrap();
println!("{}", value);
```

Before we go on further, let's separate the topic of async programming into some topics to better understand what we'll cover in this book and what we'll not cover. 

#### How \`Futures\` are implemented in the language

If you've followed the discussions about Rusts `Futures` and `async/await` you realize that there has gone a ton of work into implementing these concepts in the Runtime.



