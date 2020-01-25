![build status](https://travis-ci.com/cfsamson/books-futures-explained.svg?token=zRZ484b4roGgifn6y3ex&branch=master)

# Thought dump

The repository for the book [Future Explained in 200 Lines of Rust](https://cfsamson.github.io/books-futures-explained/)

### Data + Vtable

We need to get a basic grasp of this to be able to understand how we implement a Waker.

Next concept is `Pin`. What guarantee do we give/get?

* Not too deep into `Pin`, the real need for this trait relates to the internals of `Future`and Async/Await implementation which is another 200 lines of .... book. Interesting but really an implementation detail.
* Next book, talk about dtolnay's `await!`macro. Check if it's feasable to make our own to see what the compiler really does. Why `Pin`is needed.

### TODO:

* Find Rfc's to point to for more information about concepts. also to double check my conclusions.

Run this code in the playground:

```rust
use std::rc::Rc;

fn main() {
    use std::mem::size_of;

    println!("Size of Box<i32>: {}", size_of::<Box<i32>>());
    println!("Size of &i32: {}", size_of::<&i32>());
    println!("Size of &Box<i32>: {}", size_of::<&Box<i32>>());
    println!("Size of Box<Trait>: {}", size_of::<Box<MyTrait>>());
    println!("Size of &dyn Trait: {}", size_of::<&dyn MyTrait>());
    println!("Size of Rc<i32>: {}", size_of::<Rc<i32>>());
    println!("Size of Box<Rc<i32>>: {}", size_of::<Box<Rc<i32>>>());
    println!("Size of Rc<Trait>: {}", size_of::<Rc<MyTrait>>());
    println!("Size of &[i32]: {}", size_of::<&[i32]>());
    println!("Size of &[&dyn Trait]: {}", size_of::<&[&dyn MyTrait]>());
    println!("Size of [i32; 10]: {}", size_of::<[i32; 10]>());
    println!("Size of [&dyn Trait; 10]: {}", size_of::<[&dyn MyTrait; 10]>());
}

trait MyTrait {
    fn do_something(&self) {
        println!("See, something");
    }
}
```

### Smart pointers = Data + Vtable

### Dynamic Dispatch - short intro

For more information about this:

{% embed url="https://alschwalm.com/blog/static/2017/03/07/exploring-dynamic-dispatch-in-rust/" %}

Create our own "fat pointer". We need to do this when we create Wakers anyway so let's get to know them a little. It's probably the thing that's most difficult implementation wise.



### Bonus - pointer types

Normal pointer: just a memory location. Basic pointer type we all know \(reference/pointer\)

Fat pointer: pointer plus some other information \(slice length, a second pointer to trait information for a trait object\). 16 byte pointer, not only compiler "hint".

Smart pointer: a pointer-like type that add behaviour or restrictions eg `Box` deallocates when dropped, `Rc` tracks how many shared owner there currently are. Compiler only!

