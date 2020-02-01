# Futures Explained in 200 Lines of Rust

This book aims to explain `Futures` in Rust using an example driven approach.

The goal is to get a better understanding of `Futures` by implementing a toy
`Reactor`, a very simple `Executor` and our own `Futures`. 

We'll start off a bit differently than most other explanations. Instead of 
deferring some of the details about what's special about futures in Rust we 
try to tackle that head on first. We'll be as brief as possible, but as thorough 
as needed. This way, most question will be answered and explored up front. 

We'll end up with futures that can run an any executor like `tokio` and `async_str`.

In the end I've made some reader exercises you can do if you want to fix some
of the most glaring omissions and shortcuts we took and create a slightly better
example yourself.

> This book is developed in the open, and contributions are welcome. You'll find
> [the repository for the book itself here][book_repo]. The final example which
> you can clone, fork or copy [can be found here][example_repo]

## What does this book give you that isn't covered elsewhere?

There are many good resources and examples already. First
of all, this book will focus on `Futures` and `async/await` specifically and
not in the context of any specific runtime.

Secondly, I've always found small runnable examples very exiting to learn from. 
Thanks to [Mdbook][mdbook] the examples can even be edited and explored further. It's
all code that you can download, play with and learn from.

We'll and end up with an understandable example including a `Future`
implementation, an `Executor` and a `Reactor` in less than 200 lines of code. 
We don't rely on any dependencies or real I/O which means it's very easy to 
explore further and try your own ideas.


## Credits and thanks

I'll like to take the chance of thanking the people behind `mio`, `tokio`, 
`async_std`, `Futures`, `libc`, `crossbeam` and many other libraries which so
much is built upon. Reading and exploring some of this code is nothing less than
impressive. Even the RFCs that much of the design is built upon is written in a
way that mortal people can understand, and that requires a lot of work. So thanks!

[mdbook]: https://github.com/rust-lang/mdBook
[book_repo]: https://github.com/cfsamson/books-futures-explained
[example_repo]: https://github.com/cfsamson/examples-futures