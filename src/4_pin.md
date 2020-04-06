# Pin

> **Overview**
>
> 1. Learn how to use `Pin` and why it's required when implementing your own `Future`
> 2. Understand how to make self-referential types safe to use in Rust
> 3. Learn how borrowing across `await` points is accomplished
> 4. Get a set of practical rules to help you work with `Pin`
>
> `Pin` was suggested in [RFC#2349][rfc2349]

Let's jump strait to it. Pinning is one of those subjects which is hard to wrap
your head around in the start, but once you unlock a mental model for it
it gets significantly easier to reason about.

## Definitions

Pin is only relevant for pointers. A reference to an object is a pointer.

Pin consists of the `Pin` type and the `Unpin` marker. Pin's purpose in life is
to govern the rules that need to apply for types which implement `!Unpin`.

Yep, you're right, that's double negation right there. `!Unpin` means
"not-un-pin".

> _This naming scheme is one of Rusts safety features where it deliberately
> tests if you're too tired to safely implement a type with this marker. If
> you're starting to get confused, or even angry, by `!Unpin` it's a good sign
> that it's time to lay down the work and start over tomorrow with a fresh mind._

On a more serious note, I feel obliged to mention that there are valid reasons
for the names that were chosen. Naming is not easy, and I considered renaming
`Unpin` and `!Unpin` in this book to make them easier to reason about. 

However, an experienced member of the Rust community convinced me that that there
is just too many nuances and edge-cases to consider which is easily overlooked when
naively giving these markers different names, and I'm convinced that we'll
just have to get used to them and use them as is.

If you want to you can read a bit of the discussion from the
[internals thread][internals_unpin].

## Pinning and self-referential structs

Let's start where we left off in the last chapter by making the problem we
saw using a self-referential struct in our generator a lot simpler by making
some self-referential structs that are easier to reason about than our
state machines:

For now our example will look like this:

```rust, ignore
use std::pin::Pin;

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

Let's walk through this example since we'll be using it the rest of this chapter.

We have a self-referential struct `Test`. `Test` needs an `init` method to be
created which is strange but we'll need that to keep this example as short as
possible.

`Test` provides two methods to get a reference to the value of the fields
`a` and `b`. Since `b` is a reference to `a` we store it as a pointer since
the borrowing rules of Rust doesn't allow us to define this lifetime.

Now, let's use this example to explain the problem we encounter in detail. As
you see, this works as expected:

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    println!("a: {}, b: {}", test2.a(), test2.b());

}
# use std::pin::Pin;
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
# }
# 
# impl Test {
#     fn new(txt: &str) -> Self {
#         let a = String::from(txt);
#         Test {
#             a,
#             b: std::ptr::null(),
#         }
#     }
# 
#     // We need an `init` method to actually set our self-reference
#     fn init(&mut self) {
#         let self_ref: *const String = &self.a;
#         self.b = self_ref;
#     }
#     
#     fn a(&self) -> &str {
#         &self.a
#     } 
#     
#     fn b(&self) -> &String {
#         unsafe {&*(self.b)}
#     }
# }
```

In our main method we first instantiate two instances of `Test` and print out
the value of the fields on `test1`. We get what we'd expect:

```rust, ignore
a: test1, b: test1
a: test2, b: test2
```

Let's see what happens if we swap the data stored at the memory location
which `test1` is pointing to with the data stored at the memory location
`test2` is pointing to and vice a versa.

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, b: {}", test2.a(), test2.b());

}
# use std::pin::Pin;
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
# }
# 
# impl Test {
#     fn new(txt: &str) -> Self {
#         let a = String::from(txt);
#         Test {
#             a,
#             b: std::ptr::null(),
#         }
#     }
# 
#     fn init(&mut self) {
#         let self_ref: *const String = &self.a;
#         self.b = self_ref;
#     }
#     
#     fn a(&self) -> &str {
#         &self.a
#     } 
#     
#     fn b(&self) -> &String {
#         unsafe {&*(self.b)}
#     }
# }
```

Naively, we could think that what we should get a debug print of `test1` two
times like this

```rust, ignore
a: test1, b: test1
a: test1, b: test1
```

But instead we get:

```rust, ignore
a: test1, b: test1
a: test1, b: test2
```

The pointer to `test2.b` still points to the old location which is inside `test1`
now. The struct is not self-referential anymore, it holds a pointer to a field
in a different object. That means we can't rely on the lifetime of `test2.b` to
be tied to the lifetime of `test2` anymore.

If your still not convinced, this should at least convince you:

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    test1.a = "I've totally changed now!".to_string();
    println!("a: {}, b: {}", test2.a(), test2.b());

}
# use std::pin::Pin;
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
# }
# 
# impl Test {
#     fn new(txt: &str) -> Self {
#         let a = String::from(txt);
#         Test {
#             a,
#             b: std::ptr::null(),
#         }
#     }
# 
#     fn init(&mut self) {
#         let self_ref: *const String = &self.a;
#         self.b = self_ref;
#     }
#     
#     fn a(&self) -> &str {
#         &self.a
#     } 
#     
#     fn b(&self) -> &String {
#         unsafe {&*(self.b)}
#     }
# }
```

That shouldn't happen. There is no serious error yet, but as you can imagine
it's easy to create serious bugs using this code.

I created a diagram to help visualize what's going on:

**Fig 1: Before and after swap**
![swap_problem](./assets/swap_problem.jpg)

As you can see this results in unwanted behavior. It's easy to get this to 
segfault, show UB and fail in other spectacular ways as well.

## Pinning to the stack

Now, we can solve this problem by using `Pin` instead. Let's take a look at what
our example would look like then:

```rust, ignore
use std::pin::Pin;
use std::marker::PhantomPinned;

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
    fn init<'a>(self: Pin<&'a mut Self>) {
        let self_ptr: *const String = &self.a;
        let this = unsafe { self.get_unchecked_mut() };
        this.b = self_ptr;
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
`unsafe` if our type implements `!Unpin`.

We use the same tricks here, including requiring an `init`. If we want to fix that
and let users avoid `unsafe` we need to pin our data on the heap instead which
we'll show in a second.

Let's see what happens if we run our example now:

```rust
pub fn main() {
    // test1 is safe to move before we initialize it
    let mut test1 = Test::new("test1");
    // Notice how we shadow `test1` to prevent it from beeing accessed again
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());
     
    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
# use std::pin::Pin;
# use std::marker::PhantomPinned;
# 
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
#     _marker: PhantomPinned,
# }
# 
# 
# impl Test {
#     fn new(txt: &str) -> Self {
#         let a = String::from(txt);
#         Test {
#             a,
#             b: std::ptr::null(),
#             // This makes our type `!Unpin`
#             _marker: PhantomPinned,
#         }
#     }
#     fn init<'a>(self: Pin<&'a mut Self>) {
#         let self_ptr: *const String = &self.a;
#         let this = unsafe { self.get_unchecked_mut() };
#         this.b = self_ptr;
#     }
# 
#     fn a<'a>(self: Pin<&'a Self>) -> &'a str {
#         &self.get_ref().a
#     }
# 
#     fn b<'a>(self: Pin<&'a Self>) -> &'a String {
#         unsafe { &*(self.b) }
#     }
# }
```

Now, if we try to pull the same trick which got us in to trouble the last time
you'll get a compilation error.

```rust, compile_fail
pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());
     
    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    std::mem::swap(test1.as_mut(), test2.as_mut());
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
# use std::pin::Pin;
# use std::marker::PhantomPinned;
# 
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
#     _marker: PhantomPinned,
# }
# 
# 
# impl Test {
#     fn new(txt: &str) -> Self {
#         let a = String::from(txt);
#         Test {
#             a,
#             b: std::ptr::null(),
#             // This makes our type `!Unpin`
#             _marker: PhantomPinned,
#         }
#     }
#     fn init<'a>(self: Pin<&'a mut Self>) {
#         let self_ptr: *const String = &self.a;
#         let this = unsafe { self.get_unchecked_mut() };
#         this.b = self_ptr;
#     }
# 
#     fn a<'a>(self: Pin<&'a Self>) -> &'a str {
#         &self.get_ref().a
#     }
# 
#     fn b<'a>(self: Pin<&'a Self>) -> &'a String {
#         unsafe { &*(self.b) }
#     }
# }
```

As you see from the error you get by running the code the type system prevents
us from swapping the pinned pointers.

> It's important to note that stack pinning will always depend on the current
> stack frame we're in, so we can't create a self referential object in one
> stack frame and return it since any pointers we take to "self" is invalidated.
> 
> It also puts a lot of responsibility in your hands if you pin a value to the
> stack. A mistake that is easy to make is, forgetting to shadow the original variable 
> since you could drop the pinned pointer and access the old value
> after it's initialized like this:
>
> ```rust
> fn main() {
>    let mut test1 = Test::new("test1");
>    let mut test1_pin = unsafe { Pin::new_unchecked(&mut test1) };
>    Test::init(test1_pin.as_mut());
>    drop(test1_pin);
>    
>    let mut test2 = Test::new("test2");
>    mem::swap(&mut test1, &mut test2);
>    println!("Not self referential anymore: {:?}", test1.b);
> }
> # use std::pin::Pin;
> # use std::marker::PhantomPinned;
> # use std::mem;
> # 
> # #[derive(Debug)]
> # struct Test {
> #     a: String,
> #     b: *const String,
> #     _marker: PhantomPinned,
> # }
> # 
> # 
> # impl Test {
> #     fn new(txt: &str) -> Self {
> #         let a = String::from(txt);
> #         Test {
> #             a,
> #             b: std::ptr::null(),
> #             // This makes our type `!Unpin`
> #             _marker: PhantomPinned,
> #         }
> #     }
> #     fn init<'a>(self: Pin<&'a mut Self>) {
> #         let self_ptr: *const String = &self.a;
> #         let this = unsafe { self.get_unchecked_mut() };
> #         this.b = self_ptr;
> #     }
> # 
> #     fn a<'a>(self: Pin<&'a Self>) -> &'a str {
> #         &self.get_ref().a
> #     }
> # 
> #     fn b<'a>(self: Pin<&'a Self>) -> &'a String {
> #         unsafe { &*(self.b) }
> #     }
> # }
> ```

## Pinning to the heap

For completeness let's remove some unsafe and the need for an `init` method
at the cost of a heap allocation. Pinning to the heap is safe so the user
doesn't need to implement any unsafe code:

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

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

pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test2 = Test::new("test2");

    println!("a: {}, b: {}",test1.as_ref().a(), test1.as_ref().b());
    println!("a: {}, b: {}",test2.as_ref().a(), test2.as_ref().b());
}
```

The fact that it's safe to pin a heap allocated value even if it is `!Unpin`
makes sense. Once the data is allocated on the heap it will have a stable address.

There is no need for us as users of the API to take special care and ensure
that the self-referential pointer stays valid.

There are ways to safely give some guarantees on stack pinning as well, but right
now you need to use a crate like [pin_project][pin_project] to do that.

## Practical rules for Pinning

1. If `T: Unpin` (which is the default), then `Pin<'a, T>` is entirely
equivalent to `&'a mut T`. in other words: `Unpin` means it's OK for this type
to be moved even when pinned, so `Pin` will have no effect on such a type.

2. Getting a `&mut T` to a pinned T requires unsafe if `T: !Unpin`. In
other words: requiring a pinned pointer to a type which is `!Unpin` prevents
the _user_ of that API from moving that value unless it choses to write `unsafe`
code.

3. Pinning does nothing special with memory allocation like putting it into some
"read only" memory or anything fancy. It only uses the type system to prevent
certain operations on this value.

1. Most standard library types implement `Unpin`. The same goes for most
"normal" types you encounter in Rust. `Futures` and `Generators` are two
exceptions.

5. The main use case for `Pin` is to allow self referential types, the whole
justification for stabilizing them was to allow that. There are still corner
cases in the API which are being explored.

6. The implementation behind objects that are `!Unpin` is most likely unsafe.
Moving such a type after it has been pinned can cause the universe to crash. As of the time of writing
this book, creating and reading fields of a self referential struct still requires `unsafe`
(the only way to do it is to create a struct containing raw pointers to itself).

7. You can add a `!Unpin` bound on a type on nightly with a feature flag, or
by adding `std::marker::PhantomPinned` to your type on stable.

8. You can either pin a value to memory on the stack or on the heap.

9. Pinning a `!Unpin` pointer to the stack requires `unsafe`

10. Pinning a `!Unpin` pointer to the heap does not require `unsafe`. There is a shortcut for doing this using `Box::pin`.

> Unsafe code does not mean it's literally "unsafe", it only relieves the 
> guarantees you normally get from the compiler. An `unsafe` implementation can 
> be perfectly safe to do, but you have no safety net.


### Projection/structural pinning

In short, projection is a programming language term. `mystruct.field1` is a
projection. Structural pinning is using `Pin` on fields. This has several
caveats and is not something you'll normally see so I refer to the documentation
for that.

### Pin and Drop

The `Pin` guarantee exists from the moment the value is pinned until it's dropped.
In the `Drop` implementation you take a mutable reference to `self`, which means
extra care must be taken when implementing `Drop` for pinned types.

## Putting it all together

This is exactly what we'll do when we implement our own `Futures` stay tuned, 
we're soon finished.

[rfc2349]: https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md
[pin_project]: https://docs.rs/pin-project/
[internals_unpin]: https://internals.rust-lang.org/t/naming-pin-anchor-move/6864/12

## Bonus section: Fixing our self-referential generator and learning more about Pin

But now, let's prevent this problem using `Pin`. We'll discuss
`Pin` more in the next chapter, but you'll get an introduction here by just
reading the comments.

```rust
#![feature(optin_builtin_traits, negative_impls)] // needed to implement `!Unpin`
use std::pin::Pin;

pub fn main() {
    let gen1 = GeneratorA::start();
    let gen2 = GeneratorA::start();
    // Before we pin the pointers, this is safe to do
    // std::mem::swap(&mut gen, &mut gen2);

    // constructing a `Pin::new()` on a type which does not implement `Unpin` is
    // unsafe. A value pinned to heap can be constructed while staying in safe
    // Rust so we can use that to avoid unsafe. You can also use crates like
    // `pin_utils` to pin to the stack safely, just remember that they use
    // unsafe under the hood so it's like using an already-reviewed unsafe
    // implementation.

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

    // This won't work:
    // std::mem::swap(&mut gen, &mut gen2);
    // This will work but will just swap the pointers so nothing bad happens here:
    // std::mem::swap(&mut pinned1, &mut pinned2);

    let _ = pinned1.as_mut().resume();
    let _ = pinned2.as_mut().resume();
}

enum GeneratorState<Y, R> {
    Yielded(Y),  
    Complete(R), 
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
        borrowed: *const String,
    },
    Exit,
}

impl GeneratorA {
    fn start() -> Self {
        GeneratorA::Enter
    }
}

// This tells us that the underlying pointer is not safe to move after pinning.
// In this case, only we as implementors "feel" this, however, if someone is
// relying on our Pinned pointer this will prevent them from moving it. You need
// to enable the feature flag `#![feature(optin_builtin_traits)]` and use the
// nightly compiler to implement `!Unpin`. Normally, you would use
// `std::marker::PhantomPinned` to indicate that the struct is `!Unpin`.
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
                *this = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};

                // Trick to actually get a self reference. We can't reference
                // the `String` earlier since these references will point to the
                // location in this stack frame which will not be valid anymore
                // when this function returns.
                if let GeneratorA::Yield1 {to_borrow, borrowed} = this {
                    *borrowed = to_borrow;
                }

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