# Generators

>**Relevant for:**
>
>- Understanding how the async/await syntax works since it's how `await` is implemented
>- Knowing why we need `Pin`
>- Understanding why Rusts async model is very efficient
>
>The motivation for `Generators` can be found in [RFC#2033][rfc2033]. It's very
>well written and I can recommend reading through it (it talks as much about
>async/await as it does about generators).

The second difficult part is understanding Generators and the `Pin` type. Since
they're related we'll start off by exploring generators first. By doing that
we'll soon get to see why we need to be able to "pin" some data to a fixed
location in memory and get an introduction to `Pin` as well.

Basically, there were three main options discussed when designing how Rust would
handle concurrency:

1. Stackful coroutines, better known as green threads.
2. Using combinators.
3. Stackless coroutines, better known as generators.

### Stackful coroutines/green threads

I've written about green threads before. Go check out 
[Green Threads Explained in 200 lines of Rust][greenthreads] if you're interested.

Green threads uses the same mechanism as an OS does by creating a thread for
each task, setting up a stack, save the CPU's state and jump from one
task(thread) to another by doing a "context switch".

We yield control to the scheduler (which is a central part of the runtime in
such a system) which then continues running a different task.

Rust had green threads once, but they were removed before it hit 1.0. The state
of execution is stored in each stack so in such a solution there would be no need
for `async`, `await`, `Futures` or `Pin`. All this would be implementation details for the library.

### Combinators

`Futures 1.0` used combinators. If you've worked with `Promises` in JavaScript,
you already know combinators. In Rust they look like this:

```rust,noplaypen,ignore
let future = Connection::connect(conn_str).and_then(|conn| {
    conn.query("somerequest").map(|row|{
        SomeStruct::from(row)
    }).collect::<Vec<SomeStruct>>()
});

let rows: Result<Vec<SomeStruct>, SomeLibraryError> = block_on(future).unwrap();

```

While an effective solution there are mainly three downsides I'll focus on:

1. The error messages produced could be extremely long and arcane
2. Not optimal memory usage
3. Did not allow to borrow across combinator steps.

Point #3, is actually a major drawback with `Futures 1.0`.

Not allowing borrows across suspension points ends up being very
un-ergonomic and to accomplish some tasks it requires extra allocations or
copying which is inefficient.

The reason for the higher than optimal memory usage is that this is basically
a callback-based approach, where each closure stores all the data it needs
for computation. This means that as we chain these, the memory required to store
the needed state increases with each added step.

### Stackless coroutines/generators

This is the model used in Rust today. It has a few notable advantages:

1. It's easy to convert normal Rust code to a stackless coroutine using using
async/await as keywords (it can even be done using a macro).
2. No need for context switching and saving/restoring CPU state
3. No need to handle dynamic stack allocation
4. Very memory efficient
5. Allows us to borrow across suspension points

The last point is in contrast to `Futures 1.0`. With async/await we can do this:

```rust, ignore
async fn myfn() {
    let text = String::from("Hello world");
    let borrowed = &text[0..5];
    somefuture.await;
    println!("{}", borrowed);
}
```

Generators in Rust are implemented as state machines. The memory footprint of a
chain of computations is only defined by the largest footprint of any single
step require. That means that adding steps to a chain of computations might not
require any increased memory at all.

## How generators work

In Nightly Rust today you can use the `yield` keyword. Basically using this
keyword in a closure, converts it to a generator. A closure could look like this
before we had a concept of `Pin`:


```rust,noplaypen,ignore
#![feature(generators, generator_trait)]
use std::ops::{Generator, GeneratorState};

fn main() {
    let a: i32 = 4;
    let mut gen = move || {
        println!("Hello");
        yield a * 2;
        println!("world!");
    };

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}
```

Early on, before there was a consensus about the design of `Pin`, this
compiled to something looking similar to this:

```rust
fn main() {
    let mut gen = GeneratorA::start(4);

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}

// If you've ever wondered why the parameters are called Y and R the naming from
// the original rfc most likely holds the answer
enum GeneratorState<Y, R> {
    Yielded(Y),  // originally called `Yield(Y)`
    Complete(R), // originally called `Return(R)`
}

trait Generator {
    type Yield;
    type Return;
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter(i32),
    Yield1(i32),
    Exit,
}

impl GeneratorA {
    fn start(a1: i32) -> Self {
        GeneratorA::Enter(a1)
    }
}

impl Generator for GeneratorA {
    type Yield = i32;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
        match std::mem::replace(&mut *self, GeneratorA::Exit) {
            GeneratorA::Enter(a1) => {

          /*|---code before yield---|*/
          /*|*/ println!("Hello"); /*|*/ 
          /*|*/ let a = a1 * 2;    /*|*/
          /*|------------------------|*/

                *self = GeneratorA::Yield1(a);
                GeneratorState::Yielded(a)
            }
            GeneratorA::Yield1(_) => {

          /*|----code after yield----|*/
          /*|*/ println!("world!"); /*|*/ 
          /*|-------------------------|*/

                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}

```

>The `yield` keyword was discussed first in [RFC#1823][rfc1823] and in [RFC#1832][rfc1832].

Now that you know that the `yield` keyword in reality rewrites your code to become a state machine,
you'll also know the basics of how `await` works. It's very similar.

Now, there are some limitations in our naive state machine above. What happens when you have a 
`borrow` across a `yield` point?

We could forbid that, but **one of the major design goals for the async/await syntax has been
to allow this**. These kinds of borrows were not possible using `Futures 1.0` so we can't let this
limitation just slip and call it a day yet.

Instead of discussing it in theory, let's look at some code.

> We'll use the optimized version of the state machines which is used in Rust today. For a more
> in depth explanation see [Tyler Mandry's excellent article: How Rust optimizes async/await][optimizing-await]

```rust,noplaypen,ignore
let mut gen = move || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };
```

Now what does our rewritten state machine look like with this example?

```rust,compile_fail
# // If you've ever wondered why the parameters are called Y and R the naming from
# // the original rfc most likely holds the answer
# enum GeneratorState<Y, R> {
#     // originally called `CoResult`
#     Yielded(Y),  // originally called `Yield(Y)`
#     Complete(R), // originally called `Return(R)`
# }
# 
# trait Generator {
#     type Yield;
#     type Return;
#     fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>;
# }

enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: &String, // uh, what lifetime should this have?
    },
    Exit,
}

# impl GeneratorA {
#     fn start() -> Self {
#         GeneratorA::Enter
#     }
# }

impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
        match std::mem::replace(&mut *self, GeneratorA::Exit) {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow;
                let res = borrowed.len();

                *self = GeneratorA::Yield1 {to_borrow, borrowed};
                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {to_borrow, borrowed} => {
                println!("Hello {}", borrowed);
                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
```

If you try to compile this you'll get an error (just try it yourself by pressing play).

What is the lifetime of `&String`. It's not the same as the lifetime of `Self`. It's not `static`.
Turns out that it's not possible for us in Rusts syntax to describe this lifetime, which means, that
to make this work, we'll have to let the compiler know that _we_ control this correctly ourselves.

That means turning to unsafe.

Let's try to write an implementation that will compiler using `unsafe`. As you'll
see we end up in a _self referential struct_. A struct which holds references
into itself.

As you'll notice, this compiles just fine!

```rust,editable
pub fn main() {
    let mut gen = GeneratorA::start();
    let mut gen2 = GeneratorA::start();

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }
    
    // If you uncomment this, very bad things can happen. This is why we need `Pin`
    // std::mem::swap(&mut gen, &mut gen2);

    if let GeneratorState::Yielded(n) = gen2.resume() {
        println!("Got value {}", n);
    }

    // if you uncomment `mem::swap`.. this should now start gen2.
    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}

enum GeneratorState<Y, R> {
    Yielded(Y),  // originally called `Yield(Y)`
    Complete(R), // originally called `Return(R)`
}

trait Generator {
    type Yield;
    type Return;
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: *const String, // Normally you'll see `std::ptr::NonNull` used instead of *ptr
    },
    Exit,
}

impl GeneratorA {
    fn start() -> Self {
        GeneratorA::Enter
    }
}
impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
            match self {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow;
                let res = borrowed.len();

                // Trick to actually get a self reference
                *self = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};
                match self {
                 GeneratorA::Yield1{to_borrow, borrowed} => *borrowed = to_borrow,
                 _ => unreachable!(),  
                };

                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {borrowed, ..} => {
                let borrowed: &String = unsafe {&**borrowed};
                println!("{} world", borrowed);
                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
```

> Try to uncomment the line with `mem::swap` and see the results.

While the example above compiles just fine, we expose consumers of this this API 
to both possible undefined behavior and other memory errors while using just safe
Rust. This is a big problem!

But now, let's prevent this problem using `Pin`. We'll discuss
`Pin` more in the next chapter, but you'll get an introduction here by just
reading the comments.

```rust,editable
#![feature(optin_builtin_traits)] // needed to implement `!Unpin`
use std::pin::Pin;

pub fn main() {
    let gen1 = GeneratorA::start();
    let gen2 = GeneratorA::start();
    // Before we pin the pointers, this is safe to do
    // std::mem::swap(&mut gen, &mut gen2);

    // constructing a `Pin::new()` on a type which does not implement `Unpin` is unsafe.
    // However, as you'll see in the start of the next chapter value pinned to
    // heap can be constructed while staying in safe Rust so we can use
    // that to avoid unsafe. You can also use crates like `pin_utils` to do 
    // this safely, just remember that they use unsafe under the hood so it's 
    // like using an already-reviewed unsafe implementation.

    let mut pinned1 = Box::pin(gen1);
    let mut pinned2 = Box::pin(gen2);
    // Uncomment these if you think it's safe to pin the values to the stack instead 
    // (it is in this case). Remember to comment out the two previous lines first.
    //let mut pinned1 = unsafe { Pin::new_unchecked(&mut gen1) };
    //let mut pinned2 = unsafe { Pin::new_unchecked(&mut gen2) };

    if let GeneratorState::Yielded(n) = pinned1.as_mut().resume() {
        println!("Gen1 got value {}", n);
    }
    
    if let GeneratorState::Yielded(n) = pinned2.as_mut().resume() {
        println!("Gen2 got value {}", n);
    };

    // This won't work
    // std::mem::swap(&mut gen, &mut gen2);
    // This will work but will just swap the pointers. Nothing inherently bad happens here.
    // std::mem::swap(&mut pinned1, &mut pinned2);

    let _ = pinned1.as_mut().resume();
    let _ = pinned2.as_mut().resume();
}

enum GeneratorState<Y, R> {
    // originally called `CoResult`
    Yielded(Y),  // originally called `Yield(Y)`
    Complete(R), // originally called `Return(R)`
}

trait Generator {
    type Yield;
    type Return;
    fn resume(self: Pin<&mut Self>) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: *const String, // Normally you'll see `std::ptr::NonNull` used instead of *ptr
    },
    Exit,
}

impl GeneratorA {
    fn start() -> Self {
        GeneratorA::Enter
    }
}

// This tells us that the underlying pointer is not safe to move after pinning. In this case,
// only we as implementors "feel" this, however, if someone is relying on our Pinned pointer
// this will prevent them from moving it. You need to enable the feature flag 
// `#![feature(optin_builtin_traits)]` and use the nightly compiler to implement `!Unpin`.
// Normally, you would use `std::marker::PhantomPinned` to indicate that the
// struct is `!Unpin`.
impl !Unpin for GeneratorA { }

impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(self: Pin<&mut Self>) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
        let this = unsafe { self.get_unchecked_mut() };
            match this {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow;
                let res = borrowed.len();

                // Trick to actually get a self reference. We can't reference
                // the `String` earlier since these references will point to the
                // location in this stack frame which will not be valid anymore
                // when this function returns.
                *this = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};
                match this {
                 GeneratorA::Yield1{to_borrow, borrowed} => *borrowed = to_borrow,
                 _ => unreachable!(),  
                };

                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {borrowed, ..} => {
                let borrowed: &String = unsafe {&**borrowed};
                println!("{} world", borrowed);
                *this = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
```

Now, as you see, the consumer of this API must either:

1. Box the value and thereby allocating it on the heap
2. Use `unsafe` and pin the value to the stack. The user knows that if they move
the value afterwards it will violate the guarantee they promise to uphold when
they did their unsafe implementation.

Hopefully, after this you'll have an idea of what happens when you use the
`yield` or `await` keywords inside an async function, and why we need `Pin` if
we want to be able to safely borrow across `yield/await` points.

## Bonus section - self referential generators in Rust today

Thanks to [PR#45337][pr45337] you can actually run code like the one in our
example in Rust today using the `static` keyword on nightly. Try it for
yourself:

>Beware that the API is changing rapidly. As I was writing this book, generators
had an API change adding support for a "resume" argument to get passed into the
generator closure.
>
>Follow the progress on the [tracking issue #4312][issue43122] for [RFC#033][rfc2033].

```rust
#![feature(generators, generator_trait)]
use std::ops::{Generator, GeneratorState};


pub fn main() {
    let gen1 = static || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };
    
    let gen2 = static || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };

    let mut pinned1 = Box::pin(gen1);
    let mut pinned2 = Box::pin(gen2);

    if let GeneratorState::Yielded(n) = pinned1.as_mut().resume(()) {
        println!("Gen1 got value {}", n);
    }
    
    if let GeneratorState::Yielded(n) = pinned2.as_mut().resume(()) {
        println!("Gen2 got value {}", n);
    };

    let _ = pinned1.as_mut().resume(());
    let _ = pinned2.as_mut().resume(());
}
```

[rfc2033]: https://github.com/rust-lang/rfcs/blob/master/text/2033-experimental-coroutines.md
[greenthreads]: https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/
[rfc1823]: https://github.com/rust-lang/rfcs/pull/1823
[rfc1832]: https://github.com/rust-lang/rfcs/pull/1832
[optimizing-await]: https://tmandry.gitlab.io/blog/posts/optimizing-await-1/
[pr45337]: https://github.com/rust-lang/rust/pull/45337/files
[issue43122]: https://github.com/rust-lang/rust/issues/43122