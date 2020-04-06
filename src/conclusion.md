# Conclusion and exercises

Congratulations. Good job! If you got this far you must have stayed with me
all the way. I hope you enjoyed the ride!

Remember that you call always leave feedback, suggest improvements or ask questions
in the [issue_tracker](https://github.com/cfsamson/books-futures-explained/issues) for this book.
I'll try my best to respond to each one of them.

I'll leave you with some suggestions for exercises if you want to explore a little further below.

Until next time!

## Reader exercises

So our implementation has taken some obvious shortcuts and could use some improvement. 
Actually digging into the code and try things yourself is a good way to learn. Here are
some good exercises if you want to explore more:

### Avoid `thread::park`

The big problem using `Thread::park` and `Thread::unpark` is that the user can access these
same methods from their own code. Try to use another method to suspend our thread and wake
it up again on our command. Some hints:

* Check out `CondVars`, here are two sources [Wikipedia][condvar_wiki] and the
docs for [`CondVar`][condvar_std]
* Take a look at crates that help you with this exact problem like [Crossbeam ](https://github.com/crossbeam-rs/crossbeam)\(specifically the [`Parker`](https://docs.rs/crossbeam/0.7.3/crossbeam/sync/struct.Parker.html)\)

### Avoid wrapping the whole `Reactor` in a mutex and pass it around

First of all, protecting the whole `Reactor` and passing it around is overkill. We're only
interested in synchronizing some parts of the information it contains. Try to refactor that
out and only synchronize access to what's really needed.

I'd encourage you to have a look at how [the async_std driver is implemented](https://github.com/async-rs/async-std/blob/master/src/net/driver/mod.rs)
and [how tokios scheduler is implemented](https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/basic_scheduler.rs) to get some inspiration.

* Do you want to pass around a reference to this information using an `Arc`?
* Do you want to make a global `Reactor` so it can be accessed from anywhere?

### Building a better exectuor

Right now, we can only run one and one future. Most runtimes has a `spawn` 
function which let's you start off a future and `await` it later so you
can run multiple futures concurrently.

As I suggested in the start of this book, visiting [@stjepan'sblog series about implementing your own executors](https://stjepang.github.io/2020/01/31/build-your-own-executor.html)
is the place I would start and take it from there.

## Further reading

There are many great resources. In addition to the RFCs and articles I've already
linked to in the book, here are some of my suggestions:

[The official Asyc book](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html)

[The async_std book](https://book.async.rs/)

[Aron Turon: Designing futures for Rust](https://aturon.github.io/blog/2016/09/07/futures-design/)

[Steve Klabnik's presentation: Rust's journey to Async/Await](https://www.infoq.com/presentations/rust-2019/)

[The Tokio Blog](https://tokio.rs/blog/2019-10-scheduler/)

[Stjepan's blog with a series where he implements an Executor](https://stjepang.github.io/)

[Jon Gjengset's video on The Why, What and How of Pinning in Rust](https://youtu.be/DkMwYxfSYNQ)

[Withoutboats blog series about async/await](https://boats.gitlab.io/blog/post/2018-01-25-async-i-self-referential-structs/)

[condvar_std]: https://doc.rust-lang.org/stable/std/sync/struct.Condvar.html
[condvar_wiki]: https://en.wikipedia.org/wiki/Monitor_(synchronization)#Condition_variables
[arcwake]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.13/futures/task/trait.ArcWake.html