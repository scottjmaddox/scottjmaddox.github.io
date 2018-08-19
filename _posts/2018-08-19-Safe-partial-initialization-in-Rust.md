---
published: true
layout: post
---
Say we have a struct, `Foo`, with multiple fields that we would like to partially initialize without resorting to using `unsafe`. We could write a procedural macro called `PartialInit`, for example, which would be invoked using `derive`:

```rust
#[derive(PartialInit)]
struct Foo {
    a: Box<bool>,
    b: Box<u32>,
}
```

And which would expand into something like the following:

```rust
impl PartialInit for Foo {
    type Uninit = PartialInitFoo<Uninit<Box<bool>>, Uninit<Box<u32>>>;
    #[inline]
    fn partial_init() -> Self::Uninit {
        PartialInitFoo {
            a: Uninit::new(),
            b: Uninit::new(),
        }
    }
}

struct PartialInitFoo<A, B> {
    a: A,
    b: B,
}

impl<A, B> PartialInitFoo<Uninit<A>, B> {
    #[inline]
    fn init_a(self, a: A) -> PartialInitFoo<A, B> {
        PartialInitFoo { a, b: self.b }
    }
}

impl<A, B> PartialInitFoo<A, Uninit<B>> {
    #[inline]
    fn init_b(self, b: B) -> PartialInitFoo<A, B> {
        PartialInitFoo { a: self.a, b }
    }
}

impl PartialInitFoo<Box<bool>, Box<u32>> {
    #[inline]
    fn finalize(self) -> Foo {
        Foo {
            a: self.a,
            b: self.b,
        }
    }
}
```

Where `Uninit` and `PartialInit` are defined as follows:

```rust
union Uninit<T> {
    init: T,
    uninit: (),
}

impl<T> Uninit<T> {
    #[inline]
    fn new() -> Self {
        Self { uninit: () }
    }
}

trait PartialInit {
    type Uninit;
    fn partial_init() -> Self::Uninit;
}
```

Note 1: Currently, due to limitations around unions in stable Rust, `Uninit` can only be defined this way in unstable Rust.
An altenative stable-friendly definition is possible using `core::mem::ManuallyDrop` and `core::mem::uninitialized`, though.

Note 2: the amount of generated source code grows linearly with the
number of fields, which is as good as we can expect.

Here's some example usage code:

```rust
let partial_foo = Foo::partial_init().init_a(Box::new(false));

// Feel free to do something that might panic, here.
// `partial_foo` has type `PartialInitFoo<Box<bool>, Uninit<Box<u32>>>`,
// which will properly drop `a: Box<bool>` on panic.
// In contrast, since `b: Uninit<Box<u32>>` is wrapped in `Uninit`,
// it will not be dropped, thus avoiding a potential security
// vulnerability.

let foo = partial_foo.init_b(Box::new(42)).finalize();
// `foo` is now a fully initialized `Foo`.
```

This looks promising! Using a little bit of boilerplate (that can be automated with
a procedural macro), we can provide a safe
interface to the bug-prone problem of partial initialization! But there's a catch...

[Unfortunately, LLVM is not able to optimize away all of the copying associated
with this approach to partial initialization](https://play.rust-lang.org/?gist=e4b00689bc21274119a7ad9a6e6beea3&version=nightly&mode=release).
So for performance-critical code, this isn't a very good solution.

Furthermore, one major reason for using partial initialization is to initialize structs
that are too large to fit on the stack, where idiomatic Rust intialization (`fn new() -> Self { ... }`)
would overflow the stack. However, since this approach to partial initializaiton
requires copying the entire struct onto the stack (multiple times), it will not
work for very large structs.

So while this pattern can be used today (either manually or through a procedural macro),
it is far from perfect. _But there's an opportunity here._ Rust could take memory safety to the next
level by providing first-class support for partial initialization with type-level safety
guarantees. By bringing support into the compiler and core library, the door opens to
guaranteeing copy elision (in debug *and* release builds),
enabling any and all use cases of partial initialization of structs and enums!

## Postscript

Once const generics are
available, this approach could potentially be extended to cover partial initialization of arrays,
making it possible to implement data structures like 
[`ArrayVec`](https://docs.rs/arrayvec/latest/arrayvec/struct.ArrayVec.html)
with little-to-no `unsafe` code, making Rust even more appealing for security-critical applications.
