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

```rust,editable
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
In the `Drop` implementation you take a mutable reference to `self`, which means
extra care must be taken when implementing `Drop` for pinned types.

## Putting it all together

This is exactly what we'll do when we implement our own `Futures` stay tuned, 
we're soon finished.

[rfc2349]: https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md
[pin_utils]: https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md
