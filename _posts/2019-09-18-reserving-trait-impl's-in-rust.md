---
published: false
---
I often find myself wanting to call a function like `my_usize.try_into::<u16>()`. This function (and similar) is definied in unstable rust through the [`TryFrom` and `TryInto` traits](https://github.com/rust-lang/rust/issues/33417). Unfortunately, stabilization of these traits is currently blocked by stabilization of [the `!` type](https://github.com/rust-lang/rust/issues/35121), because we (the royal we) want to provide default implementations of `TryFrom`/`TryInto` that return `Result<T, !>` for any types that already implement `From`/`Into`. If we stabilized `TryFrom`/`TryInto` before stabilizing the `!` type, then we would have to wait and add the blanket implementation later, once `!` is stable. But since trait implementation specialization is not (currently) allowed, adding the blanket implementation later would be a breaking change; any crates that implemented both `Into<Foo> for Bar` and `TryInto<Foo> for Bar`, for example, would no longer compile.

This is a bit of a problem. `TryFrom`/`TryInto` has been implemented in unstable since May 4, 2016. That's more than 2 years ago. And it's only blocker is our inability to reserve the desired blanket impl's for later use. This strongly suggests, to me, that we need a mechanism for reserving trait impl's. In the tracking issue for the [`TryFrom` and `TryInto` traits](https://github.com/rust-lang/rust/issues/33417), a couple of options were suggested: a) add an always-on error lint to prevent implementing both `Into<Foo> for Bar` and `TryInto<Foo> for Bar`, or b) add an unstable blanket impl `impl<T, U> TryFrom<U> for T where T: From<U>`. The former might be possible with the current lint system (I don't know), but the latter is not currently possible, because trait impl's cannot be marked as unstable. Nonetheless, it's important to note that neither of these approaches are available to crate authors. What if a crate author wants to reserve a blanket trait impl for future use? There's no way to do that other than to keep the crate unstable and warn users of the pending breaking change.

I would like to suggest that we provide a more general mechanism for reserving trait impl's. Here are a few syntactical approaches we could take:

1) `reserve` keyword

```rust
reserve impl<T, U> TryFrom<U> for T where T: From<U>;
```

or 

```rust
reserve<T, U> TryFrom<U> for T where T: From<U>;
```

2) `#[reserve]` attribute

```rust
#[reserve]
impl<T, U> TryFrom<U> for T where T: From<U>;
```

3) Implicit reservation

```rust
impl<T, U> TryFrom<U> for T where T: From<U>;
```

Regardless of the syntax, though, the semantics would be the same: the impl would be reserved for future definition; conflicting impl's would produce compiler errors. This would make it possible to stabilize the `TryFrom` and `TryInto` traits now, with the blanket impl's for `From` and `Into` reserved for later definition. This would also make it possible for crate authors to reserve blanket impl's for traits that they define.

As far as how difficult this would be to implemented in the compiler, I cannot comment, but I'm hoping someone from the compiler team can.
