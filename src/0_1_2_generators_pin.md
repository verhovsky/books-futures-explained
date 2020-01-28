# Generators and Pin

So the second difficult part that there seems to be a lot of questions about
is Generators and the `Pin` type.

## Generators


>**Relevant for:**

>- Understanding how the async/await syntax works
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

## Pin

Pin is used to allow for self referential structs. An example:

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