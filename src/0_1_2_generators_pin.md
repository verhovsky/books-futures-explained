# Generators and Pin

So the second difficult part that there seems to be a lot of questions about
is Generators and the `Pin` type.

## Generators


>**Relevant for:**
>
>- Understanding how the async/await syntax works since it's how `await` is implemented
>- Why we need `Pin`
>- Why Rusts async model is extremely efficient
>
>The motivation for `Generators` can be found in [RFC#2033][rfc2033]. It's very
>well written and I can recommend reading through it (it talks as much about
>async/await as it does about generators).

Basically, there were three main options that were discussed when Rust was 
desiging how the language would handle concurrency:

1. Stackful coroutines, better known as green threads.
2. Using combinators.
3. Stackless coroutines, better known as generators.

### Stackful coroutines/green threads

I've written about green threads before. Go check out 
[Green Threads Explained in 200 lines of Rust][greenthreads] if you're interested.

Green threads uses the same mechanisms as an OS does by creating a thread for
each task, setting up a stack and forcing the CPU to save it's state and jump
from one task(thread) to another. We yield control to the scheduler which then
continues running a different task.

Rust had green threads once, but they were removed before it hit 1.0. The state
of execution is stored in each stack so in such a solution there would be no need
for `async`, `await`, `Futures` or `Pin`. All this would be implementation
details for the library.

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
un-ergonomic and often requiring extra allocations or copying to accomplish 
some tasks which is inefficient.

The reason for the higher than optimal memory usage is that this is basically
a callback-based approach, where each closure stores all the data it needs
for computation. This means that as we chain these, the memory required to store
the needed state increases with each added step.

### Stackless coroutines/generators

This is the model used in Rust today. It a few notable advantages:

1. It's easy to convert normal Rust code to a stackless corotuine using using
async/await as keywords (it can even be done using a macro).
2. No need for context switching and saving/restoring CPU state
3. No need to handle dynamic stack allocation
4. Very memory efficient
5. Allowed for borrows across suspension points

The last point is in contrast to `Futures 1.0`. With async/await we can do this:

```rust
async fn myfn() {
    let text = String::from("Hello world");
    let borrowed = &text[0..5];
}
```

Generators are implemented as state machines. The memory footprint of a chain 
of computations is only defined by the largest footprint any single step 
requires. That means that adding steps to a chain of computations might not 
require any added memory at all.

## How generators work

In Nightly Rust today you can use the `yield` keyword. Basically using this 
keyword in a closure, converts it to a generator. A closure looking like this 
(I'm going to use the terminology that's currently in Rust):

```rust,noplaypen,ignore
let a = 4;
let b = move || {
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
    // originally called `CoResult`
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

          /*|---code before yield1---|*/
          /*|*/ println!("Hello"); /*|*/ 
          /*|*/ let a = a1 * 2;    /*|*/
          /*|------------------------|*/

                *self = GeneratorA::Yield1(a);
                GeneratorState::Yielded(a)
            }
            GeneratorA::Yield1(_) => {

          /*|----code after yield1----|*/
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
> in deapth explanation see [Tyler Mandry's execellent article: How Rust optimizes async/await][optimizing-await]

```rust,noplaypen,ignore
let a = 4;
let b = move || {
        let to_borrow = String::new("Hello");
        let borrowed = &to_borrow;
        println!("{}", borrowed);
        yield a * 2;
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
                *self = GeneratorA::Yield1 {to_borrow, borrowed};
                GeneratorState::Yielded(borrowed.len())
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
to make this work, we'll have to let the compiler know that _we_ control this correct.

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

                // Tricks to actually get a self reference
                *self = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};
                match self {
                 GeneratorA::Yield1{to_borrow, borrowed} => *borrowed = to_borrow,
                 _ => ()  
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

> Try to uncomment the line with `mem::swap` and see the result of running this code.

While the example above compiles just fine, we expose users of this code to
both possible undefined behavior and other memory errors while using just safe
Rust. This is a big problem!

But now, let's prevent the segfault from happening using `Pin`. We'll discuss
`Pin` more below, but you'll get an introduction here by just reading the
comments.

```rust,editable
#![feature(optin_builtin_traits)]
use std::pin::Pin;

pub fn main() {
    let gen1 = GeneratorA::start();
    let gen2 = GeneratorA::start();
    // Before we pin the pointers, this is safe to do
    // std::mem::swap(&mut gen, &mut gen2);

    // constructing a `Pin::new()` on a type which does not implement `Unpin` is unsafe.
    // However, as I mentioned in the start of the next chapter about `Pin` a
    // boxed type automatically implements `Unpin` so to stay in safe Rust we can use 
    // that to avoid unsafe. You can also use crates like `pin_utils` to do this safely, 
    // just remember that they use unsafe under the hood so it's like using an already-reviewed 
    // unsafe implementation.

    let mut pinned1 = Box::pin(gen1);
    let mut pinned2 = Box::pin(gen2);
    // Uncomment these if you think it's safe to pin the values to the stack instead 
    // (it is in this case). Remember to comment out the two previous lines first.
    //let mut pinned1 = unsafe { Pin::new_unchecked(&mut gen1) };
    //let mut pinned2 = unsafe { Pin::new_unchecked(&mut gen2) };

    if let GeneratorState::Yielded(n) = pinned1.as_mut().resume() {
        println!("Got value {}", n);
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
                 _ => ()  
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

Now, as you see, the user of this code must either:

1. Box the value and thereby allocating it on the heap
2. Use `unsafe` and pin the value to the stack. The user knows that if they move
the value afterwards it will violate the guarantee they promise to uphold when
they did their unsafe implementation.

Now, the code which is created and the need for `Pin` to allow for borrowing
across `yield` points should be pretty clear. 

## Pin

> **Relevant for**
>
> 1. To understand `Generators` and `Futures`
> 2. Knowing how to use `Pin` is required when implementing your own `Future`
> 3. To understand self-referential types in Rust
> 4. This is the way borrowing across `await` points is accomplished
>
> `Pin` was suggested in [RFC#2349][rfc2349]

Ping consists of the `Pin` type and the `Unpin` marker. Let's start off with some general rules:

1. Pin does nothing special, it only prevents the user of an API to violate some assumtions you make when writing your (most likely) unsafe code.
2. Most standard library types implement `Unpin`
3. `Unpin` means it's OK for this type to be moved even when pinned.
4. If you `Box` a value, that boxed value automatcally implements `Unpin`.
5. The main use case for `Pin` is to allow self referential types
6. The implementation behind objects that doens't implement `Unpin` is most likely unsafe
   1. `Pin` prevents users from your code to break the assumtions you make when writing the `unsafe` implementation
   2. It doesn't solve the fact that you'll have to write unsafe code to actually implement it
7. You're not really meant to be implementing `!Unpin`, but you can on nightly with a feature flag


> Unsafe code does not mean it's litterally "unsafe", it only relieves the 
> guarantees you normally get from the compiler. An `unsafe` implementation can 
> be perfectly safe to do, but you have no safety net.

Let's take a look at an example:

```rust,editable
use std::pin::Pin;

fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2); // try commenting out this line
    println!("a: {}, b: {}", test2.a(), test2.b());

}

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
}

impl Test {
    fn new(txt: &str) -> Self {
        let a = String::from(txt);
        Test {
            a,
            b: std::ptr::null(),
        }
    }
    
    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.b = self_ref;
    }
    
    fn a(&self) -> &str {
        &self.a
    } 
    
    fn b(&self) -> &String {
        unsafe {&*(self.b)}
    }
}
```

As you can see this results in unwanted behavior. The pointer to `b` stays the
same and points to the old value. It's easy to get this to segfault, and fail
in other spectacular ways as well.

Pin essentially prevents the **user** of your unsafe code 
(even if that means yourself) move the value after it's pinned.

If we change the example to using `Pin` instead:

```rust,editable,compile_fail
use std::pin::Pin;
use std::marker::PhantomPinned;

pub fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test1_pin = unsafe { Pin::new_unchecked(&mut test1) }; 
    let mut test2 = Test::new("test2");
    test2.init();
    let mut test2_pin = unsafe { Pin::new_unchecked(&mut test2) };

    println!(
        "a: {}, b: {}",
        Test::a(test1_pin.as_ref()),
        Test::b(test1_pin.as_ref())
    );

    // Try to uncomment this and see what happens
    // std::mem::swap(test1_pin.as_mut(), test2_pin.as_mut());
    println!(
        "a: {}, b: {}",
        Test::a(test2_pin.as_ref()),
        Test::b(test2_pin.as_ref())
    );
}

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}


impl Test {
    fn new(txt: &str) -> Self {
        let a = String::from(txt);
        Test {
            a,
            b: std::ptr::null(),
            // This makes our type `!Unpin`
            _marker: PhantomPinned,
        }
    }
    fn init(&mut self) {
        let self_ptr: *const String = &self.a;
        self.b = self_ptr;
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}

```

Now, what we've done here is pinning a stack address. That will always be
`unsafe` if our type implements `!Unpin`, in other words. That our type is not
`Unpin` which is the norm.

We use some tricks here, including requiring an `init`. If we want to fix that
and let users avoid `unsafe` we need to place our data on the heap.

Stack pinning will always depend on the current stack frame we're in, so we
can't create a self referential object in one stack frame and return it since
any pointers we take to "self" is invalidated.

The next example solves some of our friction at the cost of a heap allocation.

```rust, editbable
use std::pin::Pin;
use std::marker::PhantomPinned;

pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test2 = Test::new("test2");

    println!("a: {}, b: {}",test1.as_ref().a(), test1.as_ref().b());

    // Try to uncomment this and see what happens
    // std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, b: {}",test2.as_ref().a(), test2.as_ref().b());
}

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> {
        let a = String::from(txt);
        let t = Test {
            a,
            b: std::ptr::null(),
            _marker: PhantomPinned,
        };
        let mut boxed = Box::pin(t);
        let self_ptr: *const String = &boxed.as_ref().a;
        unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };

        boxed
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}
```

Seeing this we're ready to sum up with a few more points to remember about
pinning:

1. Pinning only makes sense to do for types that are `!Unpin`
2. Pinning a `!Unpin` pointer to the stack will requires `unsafe`
3. Pinning a boxed value will not require `unsafe`, even if the type is `!Unpin`
4. If T: Unpin (which is the default), then Pin<'a, T> is entirely equivalent to &'a mut T.
5. Getting a `&mut T` to a pinned pointer requires unsafe if `T: !Unpin`
6. Pinning is really only useful when implementing self-referential types.  
For all intents and purposes you can think of `!Unpin` = self-referential-type

The fact that boxing (heap allocating) a value that implements `!Unpin` is safe
makes sense. Once the data is allocated on the heap it will have a stable address. 

There are ways to safely give some guarantees on stack pinning as well, but right
now you need to use a crate like [pin_utils]:[pin_utils] to do that.

### Projection/structural pinning

In short, projection is using a field on your type. `mystruct.field1` is a
projection. Structural pinning is using `Pin` on struct fields. This has several
caveats and is not something you'll normally see so I refer to the documentation
for that.

### Pin and Drop

The `Pin` guarantee exists from the moment the value is pinned until it's dropped.
In the `Drop` implementation you take a mutabl reference to `self`, which means
extra care must be taken when implementing `Drop` for pinned types.

## Putting it all together

This is exactly what we'll do when we implement our own `Futures` stay tuned, 
we're soon finished.

[rfc2033]: https://github.com/rust-lang/rfcs/blob/master/text/2033-experimental-coroutines.md
[greenthreads]: https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/
[rfc1823]: https://github.com/rust-lang/rfcs/pull/1823
[rfc1832]: https://github.com/rust-lang/rfcs/pull/1832
[rfc2349]: https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md
[optimizing-await]: https://tmandry.gitlab.io/blog/posts/optimizing-await-1/
[pin_utils]: https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md