![build status](https://travis-ci.com/cfsamson/books-futures-explained.svg?branch=master)

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

## Contributing

All kinds of contributions are welcome. Spelling, wording or clarifications are
very welcome as well as adding or suggesting changes to the content. I'd appreciate
if you contribute through a PR.

Feedback, questions or discussion is welcome in the issue tracker.

## Changelog

**2020-04-06:** Final draft finished

## License

This book is MIT licensed.

[rendered]: https://cfsamson.github.io/books-futures-explained/