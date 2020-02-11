# Waker and Context

> **Relevant for:**
>
> - Understanding how the Waker object is constructed
> - Learning how the runtime know when a leaf-future can resume
> - Learning the basics of dynamic dispatch and trait objects

## The Waker

The `Waker` trait is an interface where a

One of the most confusing things we encounter when implementing our own `Futures` 
is how we implement a `Waker` . Creating a `Waker` involves creating a `vtable` 
which allows us to use dynamic dispatch to call methods on a _type erased_ trait 
object we construct our selves.

>If you want to know more about dynamic dispatch in Rust I can recommend  an article written by Adam Schwalm called [Exploring Dynamic Dispatch in Rust](https://alschwalm.com/blog/static/2017/03/07/exploring-dynamic-dispatch-in-rust/).

Let's explain this a bit more in detail.

## Fat pointers in Rust

Let's take a look at the size of some different pointer types in Rust. If we
run the following code. _(You'll have to press "play" to see the output)_:

``` rust
# use std::mem::size_of;
trait SomeTrait { }

fn main() {
    println!("======== The size of different pointers in Rust: ========");
    println!("&dyn Trait:-----{}", size_of::<&dyn SomeTrait>());
    println!("&[&dyn Trait]:--{}", size_of::<&[&dyn SomeTrait]>());
    println!("Box<Trait>:-----{}", size_of::<Box<SomeTrait>>());
    println!("&i32:-----------{}", size_of::<&i32>());
    println!("&[i32]:---------{}", size_of::<&[i32]>());
    println!("Box<i32>:-------{}", size_of::<Box<i32>>());
    println!("&Box<i32>:------{}", size_of::<&Box<i32>>());
    println!("[&dyn Trait;4]:-{}", size_of::<[&dyn SomeTrait; 4]>());
    println!("[i32;4]:--------{}", size_of::<[i32; 4]>());
}
```

As you see from the output after running this, the sizes of the references varies.
Many are 8 bytes (which is a pointer size on 64 bit systems), but some are 16
bytes.

The 16 byte sized pointers are called "fat pointers" since they carry extra
information.

**Example `&[i32]` :**

- The first 8 bytes is the actual pointer to the first element in the array (or part of an array the slice refers to)
- The second 8 bytes is the length of the slice.

**Example `&dyn SomeTrait`:**

This is the type of fat pointer we'll concern ourselves about going forward.
`&dyn SomeTrait` is a reference to a trait, or what Rust calls a _trait object_.

The layout for a pointer to a _trait object_ looks like this:

- The first 8 bytes points to the `data` for the trait object
- The second 8 bytes points to the `vtable` for the trait object

The reason for this is to allow us to refer to an object we know nothing about
except that it implements the methods defined by our trait. To accomplish this we use _dynamic dispatch_.

Let's explain this in code instead of words by implementing our own trait
object from these parts:

>This is an example of _editable_ code. You can change everything in the example
and try to run it. If you want to go back, press the undo symbol. Keep an eye
out for these as we go forward. Many examples will be editable.

```rust, editable
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
        add as usize, // function pointer - try changing the order of `add`
        sub as usize, // function pointer - and `sub` to see what happens
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

The reason we go through this will be clear later on when we implement our own
`Waker` we'll actually set up a `vtable` like we do here to and knowing what
it is will make this much less mysterious.
