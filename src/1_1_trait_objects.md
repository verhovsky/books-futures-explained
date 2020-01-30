# Trait objects and fat pointers

## Trait objects and dynamic dispatch

The single most confusing topic we encounter when implementing our own `Futures` 
is how we implement a `Waker` . Creating a `Waker` involves creating a `vtable` 
which allows using dynamic dispatch to call methods on a _type erased_ trait 
object we construct our selves.

If you want to know more about dynamic dispatch in Rust I can recommend this article:

https://alschwalm.com/blog/static/2017/03/07/exploring-dynamic-dispatch-in-rust/

Let's explain this a bit more in detail.

## Fat pointers in Rust

Let's take a look at the size of some different pointer types in Rust. If we
run the following code:

``` rust
# use std::mem::size_of;
trait SomeTrait { }

fn main() {
    println!("Size of Box<i32>: {}", size_of::<Box<i32>>());
    println!("Size of &i32: {}", size_of::<&i32>());
    println!("Size of &Box<i32>: {}", size_of::<&Box<i32>>());
    println!("Size of Box<Trait>: {}", size_of::<Box<SomeTrait>>());
    println!("Size of &dyn Trait: {}", size_of::<&dyn SomeTrait>());
    println!("Size of &[i32]: {}", size_of::<&[i32]>());
    println!("Size of &[&dyn Trait]: {}", size_of::<&[&dyn SomeTrait]>());
    println!("Size of [i32; 10]: {}", size_of::<[i32; 10]>());
    println!("Size of [&dyn Trait; 10]: {}", size_of::<[&dyn SomeTrait; 10]>());
}
```

As you see from the output after running this, the sizes of the references varies.
Most are 8 bytes (which is a pointer size on 64 bit systems), but some are 16
bytes.

The 16 byte sized pointers are called "fat pointers" since they carry more extra
information.

**In the case of `&[i32]` :** 

* The first 8 bytes is the actual pointer to the first element in the array

(or part of an array the slice refers to)

* The second 8 bytes is the length of the slice.

The one we'll concern ourselves about is the references to traits, or
_trait objects_ as they're called in Rust.

`&dyn SomeTrait` is an example of a _trait object_ 
 
 The layout for a pointer to a _trait object_ looks like this: 

* The first 8 bytes points to the `data` for the trait object
* The second 8 bytes points to the `vtable` for the trait object

The reason for this is to allow us to refer to an object we know nothing about
except that it implements the methods defined by our trait. To allow this we use
dynamic dispatch.

Let's explain this in code instead of words by implementing our own trait
object from these parts:

``` rust
// A reference to a trait object is a fat pointer: (data_ptr, vtable_ptr)
trait Test {
    fn add(&self) -> i32;
    fn sub(&self) -> i32;
    fn mul(&self) -> i32;
}

// This will represent our home brewn fat pointer to a trait object
#[repr(C)]
struct FatPointer<'a> {
    /// A reference is a pointer to an instantiated `Data` instance
    data: &'a mut Data,
    /// Since we need to pass in literal values like length and alignment it's
    /// easiest for us to convert pointers to usize-integers instead of the other way around.
    vtable: *const usize,
}

// This is the data in our trait object. It's just two numbers we want to operate on.
struct Data {
    a: i32,
    b: i32,
}

// ====== function definitions ======
fn add(s: &Data) -> i32 {
    s.a + s.b
}
fn sub(s: &Data) -> i32 {
    s.a - s.b
}
fn mul(s: &Data) -> i32 {
    s.a * s.b
}

fn main() {
    let mut data = Data {a: 3, b: 2};
    // vtable is like special purpose array of pointer-length types with a fixed
    // format where the three first values has a special meaning like the
    // length of the array is encoded in the array itself as the second value.
    let vtable = vec![
        0,            // pointer to `Drop` (which we're not implementing here)
        6,            // lenght of vtable
        8,            // alignment
        // we need to make sure we add these in the same order as defined in the Trait.
        // Try changing the order of add and sub and see what happens.
        add as usize, // function pointer
        sub as usize, // function pointer 
        mul as usize, // function pointer
    ];

    let fat_pointer = FatPointer { data: &mut data, vtable: vtable.as_ptr()};
    let test = unsafe { std::mem::transmute::<FatPointer, &dyn Test>(fat_pointer) };

    // And voalá, it's now a trait object we can call methods on
    println!("Add: 3 + 2 = {}", test.add());
    println!("Sub: 3 - 2 = {}", test.sub());
    println!("Mul: 3 * 2 = {}", test.mul());
}

```

If you run this code by pressing the "play" button at the top you'll se it
outputs just what we expect.

This code example is editable so you can change it
and run it to see what happens.

The reason we go through this will be clear later on when we implement our own
`Waker` we'll actually set up a `vtable` like we do here to and knowing what
it is will make this much less mysterious.

## Reactor/Executor pattern

If you don't know what this is, you should take a few minutes and read about
it. You will encounter the term `Reactor` and `Executor` a lot when working
with async code in Rust.

I have written a quick introduction explaining this pattern before which you
can take a look at here:


[![homepage][1]][2]

<div  style="text-align:center">
<a href="https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/appendix-1/reactor-executor-pattern">Epoll, Kqueue and IOCP Explained - The Reactor-Executor Pattern</a>
</div>

I'll re-iterate the most important parts here.

This pattern consists of at least 2 parts:

1. A reactor
    - handles some kind of event queue
    - has the responsibility of respoonding to events
2. An executor
    - Often has a scheduler
    - Holds a set of suspended tasks, and has the responsibility of resuming
    them when an event has occurred
3. The concept of a task
    - A set of operations that can be stopped half way and resumed later on

This is a pattern not only used in Rust, but it's very popular in Rust due to 
how well it separates concerns between handling and scheduling tasks, and queing
and responding to I/O events.

The only thing Rust as a language defines is the _task_. In Rust we call an 
incorruptible task a `Future`. Futures has a  well defined interface, which means
they can be used across the entire ecosystem.

In addition, Rust provides a way for the Reactor and Executor to communicate
through the `Waker`. We'll get to know these in the following chapters.

Providing these pieces let's Rust take care a lot of the ergonomic "friction"
programmers meet when faced with async code, and still not dictate any
preferred runtime to actually do the scheduling and I/O queues.

It's important to know that Rust doesn't provide a runtime, so you have to choose
one. [async std](https://github.com/async-rs/async-std) and [tokio](https://github.com/tokio-rs/tokio) are two popular ones.

With that out of the way, let's move on to our main example.




[1]: ./assets/reactorexecutor.png
[2]: https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/appendix-1/reactor-executor-pattern