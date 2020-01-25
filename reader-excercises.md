# Reader excercises

So our implementation has taken some obvious shortcuts and could use some improvement. Actually digging into the code and try things yourself is a good way to learn. Here are som relatively simple and good excersises:

### Avoid \`thread::park\`

The big problem using `Thread::park` and `Thread::unpark` is that the user can access these same methods from their own code. Try to use another method of telling the OS to supend our thread and wake it up again on our command. Some hints:

* Check out `CondVars`, here are two sources Wikipedia and the docs for `CondVar`
* Take a look at crates that help you with this exact problem like [Crossbeam ](https://github.com/crossbeam-rs/crossbeam)\(specifically the [`Parker`](https://docs.rs/crossbeam/0.7.3/crossbeam/sync/struct.Parker.html)\)

### Avoid wrapping the whole \`Reactor\` in a mutex and pass it around

First of all, protecting the whole `Reactor` and passing it around is overkill. We're only interested in synchronizing some parts of the information it contains. Try to refactor that out and only synchonize access to what's really needed.

* Do you want to pass around a reference to this information using an `Arc`?
* Do you want to make this information global so it can be accessed from anywhere?

Next , using a `Mutex` as a synchonization mechanism might be overkill since many methods only reads data. 

* Could an [`RwLock`](https://doc.rust-lang.org/stable/std/sync/struct.RwLock.html) be more efficient some places?
* Could you use any of the synchonization mechanisms in [Crossbeam](https://github.com/crossbeam-rs/crossbeam)?
* Do you want to dig into [atomics in Rust and implement a synchonization mechanism](https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/appendix-1/atomics-in-rust) of your own?

### Avoid creating a new Waker for every event

Right now we create a new instance of a Waker for every event we create. Is this really needed? 

* Could we create one instance and then cache it \(see [this article from `u/sjepang`](https://stjepang.github.io/2020/01/25/build-your-own-block-on.html)\)?
  * Should we cache it in `thread_local!` storage?
  * Or should be cache it using a global constant?

### Could we implement more methods on our executor?

What about CPU intensice tasks? Right now they'll prevent our executor thread from progressing an handling events. Could you create a thread pool and create a method to send such tasks to the threadpool instead together with a Waker which will wake up the executor thread once the CPU intensive task is done?

In both `async_std` and `tokio` this method is called `spawn_blocking`, a good place to start is to read the documentation and the code thy use to implement that.



