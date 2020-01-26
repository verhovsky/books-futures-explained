# Generators and Pin

So the second difficult part that there seems to be a lot of questions about
is Generators and the `Pin` type.

## Generators

```
**Relevant for:**

- Understanding how the async/await syntax works
- Why we need `Pin`
- Why Rusts async model is extremely efficient
```

The motivation for `Generators` can be found in [RFC#2033][rfc2033]. It's very
well written and I can recommend reading through it (it talks as much about
async/await as it does about generators).

Basically, there were three main options that were discussed when Rust was 
desiging how the language would handle concurrency:

1. Stackfull coroutines, better known as green threads.
2. Using combinators.
3. Stackless coroutines, better known as generators.


[rfc2033]: https://github.com/rust-lang/rfcs/blob/master/text/2033-experimental-coroutines.md