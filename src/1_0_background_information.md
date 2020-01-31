# Some background information

Before we start implementing our `Futures` , we'll go through some background
information that will help demystify some of the concepts we encounter.

Actually, after going through these concepts, implementing futures will seem
pretty simple. I promise.

## First things first

If you find the concepts of concurrency and async programming confusing in
general, I know where you're coming from and I have written some resources to 
try to give a high level overview that will make it easier to learn Rusts 
`Futures` afterwards:

* [Async Basics - The difference between concurrency and parallelism](https://cfsamson.github.io/book-exploring-async-basics/1_concurrent_vs_parallel.html)
* [Async Basics - Async history](https://cfsamson.github.io/book-exploring-async-basics/2_async_history.html)
* [Async Basics - Strategies for handling I/O](https://cfsamson.github.io/book-exploring-async-basics/5_strategies_for_handling_io.html)
* [Async Basics - Epoll, Kqueue and IOCP](https://cfsamson.github.io/book-exploring-async-basics/6_epoll_kqueue_iocp.html)

Now learning these concepts by studying futures is making it much harder than
it needs to be, so go on and read these chapters. I'll be right here when
you're back. 

However, if you feel that you have the basics covered, then go right on. Let's
get moving!