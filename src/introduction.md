# Futures Explained in 200 Lines of Rust

This book aims to explain `Futures` in Rust using an example driven approach,
exploring why they're designed the way they are, and how they work. We'll also
take a look at some of the alternatives we have when dealing with concurrency
in programming.

Going into the level of detail I do in this book is not needed to use futures
or async/await in Rust. It's for the curious out there that want to know _how_
it all works.

## What this book covers

This book will try to explain everything you might wonder about up until the
topic of different types of executors and runtimes. We'll just implement a very
simple runtime in this book introducing some concepts but it's enough to get
started.

[Stjepan Glavina](https://github.com/stjepang) has made an excellent series of
articles about async runtimes and executors, and if the rumors are right there
is more to come from him in the near future.

The way you should go about it is to read this book first, then continue
reading the [articles from stejpang](https://stjepang.github.io/) to learn more
about runtimes and how they work, especially:

1. [Build your own block_on()](https://stjepang.github.io/2020/01/25/build-your-own-block-on.html)
2. [Build your own executor](https://stjepang.github.io/2020/01/31/build-your-own-executor.html)

I've limited myself to a 200 line main example (hence the title) to limit the
scope and introduce an example that can easily be explored further.

However, there is a lot to digest and it's not what I would call easy, but we'll
take everything step by step so get a cup of tea and relax. 

I hope you enjoy the ride.

> This book is developed in the open, and contributions are welcome. You'll find
> [the repository for the book itself here][book_repo]. The final example which
> you can clone, fork or copy [can be found here][example_repo]. Any suggestions
> or improvements can be filed as a PR or in the issue tracker for the book.
>
> As always, all kinds of feedback is welcome.

## Reader exercises and further reading

In the last [chapter](conclusion.md) I've taken the liberty to suggest some
small exercises if you want to explore a little further.

This book is also the fourth book I have written about concurrent programming
in Rust. If you like it, you might want to check out the others as well:

- [Green Threads Explained in 200 lines of rust](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/)
- [The Node Experiment - Exploring Async Basics with Rust](https://cfsamson.github.io/book-exploring-async-basics/)
- [Epoll, Kqueue and IOCP Explained with Rust](https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/)

## Credits and thanks

I'll like to take the chance of thanking the people behind `mio`, `tokio`, 
`async_std`, `Futures`, `libc`, `crossbeam` which so underpins so much of the
async ecosystem and and rarely gets enough praise.

A special thanks to [jonhoo](https://github.com/jonhoo) who was kind enough to
give me some valuable feedback on a very early draft of this book. He has not
read the finished product, but a thanks is definitely due.

[mdbook]: https://github.com/rust-lang/mdBook
[book_repo]: https://github.com/cfsamson/books-futures-explained
[example_repo]: https://github.com/cfsamson/examples-futures