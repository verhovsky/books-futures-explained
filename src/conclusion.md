# Conclusion and exercises

Congratulations. Good job! If you got this far you must have stayed with me
all the way. I hope you enjoyed the ride!

I'll leave you with some predictions and a set of exercises I'm suggesting for
those interested.

Futures will be more ergonomic to use with time. For example, instead of having
to create a `RawWaker` and so on, the `Waker` will also be possible to implement
as a normal `Trait`. It's probably going to be pretty similar to
[ArcWake][arcwake].

There will probably be several more improvements like this, but since relatively
few people will actually need implement leaf Futures compared to those that use
them, focus will first and foremost be on how ergonomic it's to work with
futures inside async/await functions and blocks.

It will still take some time for the ecosystem to migrate over to `Futures 3.0`
but since the advantages are so huge, it will not be a split between libraries
using `Futures 1.0` and libraries using `Futures 3.0` for long.

# Reader exercises

So our implementation has taken some obvious shortcuts and could use some improvement. Actually digging into the code and try things yourself is a good
way to learn. Here are some good exercises if you want to explore more:

## Avoid `thread::park`

The big problem using `Thread::park` and `Thread::unpark` is that the user can access these same methods from their own code. Try to use another method to
suspend our thread and wake it up again on our command. Some hints:

* Check out `CondVars`, here are two sources [Wikipedia][condvar_wiki] and the
docs for [`CondVar`][condvar_std]
* Take a look at crates that help you with this exact problem like [Crossbeam ](https://github.com/crossbeam-rs/crossbeam)\(specifically the [`Parker`](https://docs.rs/crossbeam/0.7.3/crossbeam/sync/struct.Parker.html)\)

## Avoid wrapping the whole `Reactor` in a mutex and pass it around

First of all, protecting the whole `Reactor` and passing it around is overkill. We're only interested in synchronizing some parts of the information it contains. Try to refactor that out and only synchronize access to what's really needed.

* Do you want to pass around a reference to this information using an `Arc`?
* Do you want to make a global `Reactor` so it can be accessed from anywhere?

Next , using a `Mutex` as a synchronization mechanism might be overkill since many methods only reads data. 

* Could an [`RwLock`](https://doc.rust-lang.org/stable/std/sync/struct.RwLock.html) be more efficient some places?
* Could you use any of the synchronization mechanisms in [Crossbeam](https://github.com/crossbeam-rs/crossbeam)?
* Do you want to dig into [atomics in Rust and implement a synchronization mechanism](https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/appendix-1/atomics-in-rust) of your own?

## Avoid creating a new Waker for every event

Right now we create a new instance of a Waker for every event we create. Is this really needed? 

* Could we create one instance and then cache it \(see [this article from `u/sjepang`](https://stjepang.github.io/2020/01/25/build-your-own-block-on.html)\)?
  * Should we cache it in `thread_local!` storage?
  * Or should be cache it using a global constant?

## Could we implement more methods on our executor?

What about CPU intensive tasks? Right now they'll prevent our executor thread from progressing an handling events. Could you create a thread pool and create a method to send such tasks to the thread pool instead together with a Waker which will wake up the executor thread once the CPU intensive task is done?

In both `async_std` and `tokio` this method is called `spawn_blocking`, a good place to start is to read the documentation and the code thy use to implement that.

## Building a better exectuor

Right now, we can only run one and one future. Most runtimes has a `spawn` 
function which let's you start off a future and `await` it later so you
can run multiple futures concurrently.

As I'm writing this [@stjepan](https://github.com/stjepang) is writing a blog
series about implementing your own executors, and he just released a post
on how to accomplish just this you can visit [here](https://stjepang.github.io/2020/01/31/build-your-own-executor.html).
He knows what he's talking about so I recommend following that.

In the [bonus_spawn](https://github.com/cfsamson/examples-futures/tree/bonus_spawn) 
branch of the example repository you can also find an extremely simplified 
(and worse) way of accomplishing the same in only a few lines of code.

## Further reading

There are many great resources for further study. In addition to the RFCs and
articles I've already linked to in the book, here are some of my suggestions:

[The official Asyc book](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html)

[The async_std book](https://book.async.rs/)

[Aron Turon: Designing futures for Rust](https://aturon.github.io/blog/2016/09/07/futures-design/)

[Steve Klabnik's presentation: Rust's journey to Async/Await](https://www.infoq.com/presentations/rust-2019/)

[The Tokio Blog](https://tokio.rs/blog/2019-10-scheduler/)

[Stjepan's blog with a series where he implements an Executor](https://stjepang.github.io/)

[Jon Gjengset's video on The Why, What and How of Pinning in Rust](https://youtu.be/DkMwYxfSYNQ)

[condvar_std]: https://doc.rust-lang.org/stable/std/sync/struct.Condvar.html
[condvar_wiki]: https://en.wikipedia.org/wiki/Monitor_(synchronization)#Condition_variables
[arcwake]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.13/futures/task/trait.ArcWake.html