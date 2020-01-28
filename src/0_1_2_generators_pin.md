# Generators and Pin

So the second difficult part that there seems to be a lot of questions about
is Generators and the `Pin` type.

## Generators


>**Relevant for:**

>- Understanding how the async/await syntax works, and how they're implemented
>- Why we need `Pin`
>- Why Rusts async model is extremely efficient

The motivation for `Generators` can be found in [RFC#2033][rfc2033]. It's very
well written and I can recommend reading through it (it talks as much about
async/await as it does about generators).

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
While an effective solution there are mainly two downsides I'll focus on:

1. The error messages produced could be extremely long and arcane
2. Not optimal memory usage

The reason for the higher than optimal memory usage is that this is basically
a callback-based approach, where each closure stores all the data it needs
for computation. This means that as we chain these, the memory required to store
the needed state increases with each added step.

### Stackless coroutines/generators

This is the model used in Async/Await today. It has two advantages:

1. It's easy to convert normal Rust code to a stackless corotuine using using
async/await as keywords (it can even be done using a macro).
2. It uses memory very efficiently

The second point is in contrast to `Futures 1.0` (well, both are efficient in
practice but thats beside the point). Generators are implemented as state
machines. The memory footprint of a chain of computations is only defined by
the largest footprint any single step requires. That means that adding steps to
a chain of computations might not require any added memory at all.

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

We could forbid that, but **one of the major design goals** for the async/await syntax has been
to allow this. These kinds of borrows were not possible using `Futures 1.0` so we can't let this
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
to make this work, we'll have to let the compiler know that _we_ control this correctlt.

That means turning to unsafe.

Now, as you'll notice, this compiles:

```rust
pub fn test2() {
    let mut gen = GeneratorA::start();

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    let mut gen2 = GeneratorA::start();
    // If you uncomment this, very bad things can happen. This is why we need `Pin`
    // let mut gen2 = GeneratorA::start();
    //std::mem::swap(&mut gen, &mut gen2);

    if let GeneratorState::Complete(()) = gen2.resume() {
        ()
    };
}

use std::ptr::NonNull;

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

            GeneratorA::Yield1 {to_borrow, borrowed} => {
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

But now, let's

```rust
```

However, this is also the point where we need to talk about one more concept to 

## Pin

> Why
>
> 1. To understand `Generators` and `Futures`
> 2. Knowing how to use `Pin` when implementing your own `Future`
> 3. Understand self-referential types in Rust
>
> `Pin` was suggested in [RFC#2349][rfc2349]

Ping consists of the `Pin` type and the `Unpin` marker. Let's start off with some general rules:

1. Pin does nothing special, it only prevents the user of an API to violate some assumtions you make when writing your (most likely) unsafe code.
2. Most standard library types implement `Unpin`
3. `Unpin` means it's OK for this type.
4. If you `Box` a value, that boxed value automatcally implements `Unpin`.
5. The absolute main use case for `Pin` is to allow self referential types
6. The implementation behind objects that doens't implement `Unpin` is always unsafe
   1. `Pin` prevents users from your code to break the assumtions you make when writing the `unsafe` implementation
   2. It doesn't solve the fact that you'll have to write unsafe code to actually implement it

To get a 

> Unsafe code does not mean it's litterally "unsafe", it only relieves the guarantees you normally get from the compiler.
> An `unsafe` implementation can be perfectly safe to do, but you have no safety net.

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
(even if that means yourself) to do such actions.

If we change the example to using `Pin` instead:

```rust,editable,compile_fail
use std::pin::Pin;

fn main() {
    let mut test1 = Test::new("test1");
    let mut test1_pin = test1.init();
    let mut test2 = Test::new("test2");
    let mut test2_pin = test2.init();

    println!("a: {}, b: {}", test1_pin.as_ref().a(), test1_pin.as_ref().b());

    // try fixing as the compiler suggests. Is there any `swap` happening?
    // Look closely at the printout.
    std::mem::swap(test1_pin.as_mut(), test2_pin.as_mut());
    println!("a: {}, b: {}", test2_pin.as_ref().a(), test2_pin.as_ref().b());

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
    
    fn init(&mut self) -> Pin<&mut Self> {
        let self_ptr: *const String = &self.a;
        self.b = self_ptr;
        Pin::new(self)
    }
    
    fn a(&self) -> &str {
        &self.a
    } 
    
    fn b(&self) -> &String {
        unsafe {&*(self.b)}
    }
}

```

However, to get to know the normal way of implementing such an API which is what we'll see going
forward, we can rewrite the code above into this:

```rust, editbable, compile_fail
use std::pin::Pin;

pub fn test1() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test1_pin = Pin::new(&mut test1); 
    let mut test2 = Test::new("test2");
    test2.init();
    let mut test2_pin = Pin::new(&mut test2);

    println!(
        "a: {}, b: {}",
        Test::a(test1_pin.as_ref()),
        Test::b(test1_pin.as_ref())
    );

    // try fixing as the compiler suggests. Is there any `swap` happening?
    // Look closely at the printout.
    std::mem::swap(test1_pin.as_mut(), test2_pin.as_mut());
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

There is one caviat here. Our struct `Test` implements `Unpin`. Now this will be the "normal case"
since most types implement `Unpin`. However, a type which 

## Putting it all together

Now that we've seen how `Pin` works 

pinning
```ignore
// If we borrow through yield points, we end up with this error

  --> src\main.rs:12:11
   |
5  |     match gen.resume() {
   |           --- first mutable borrow occurs here
...
12 |     match gen.resume() {
   |           ^^^
   |           |
   |           second mutable borrow occurs here
   |           first borrow later used here
```

[rfc2033]: https://github.com/rust-lang/rfcs/blob/master/text/2033-experimental-coroutines.md
[greenthreads]: https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/
[rfc1823]: https://github.com/rust-lang/rfcs/pull/1823
[rfc1832]: https://github.com/rust-lang/rfcs/pull/1832
[rfc2349]: https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md
[optimizing-await]: https://tmandry.gitlab.io/blog/posts/optimizing-await-1/