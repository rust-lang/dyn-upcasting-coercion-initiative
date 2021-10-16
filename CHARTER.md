# Dyn upcast Charter

## Goals

### Summary and problem statement

* Add the `trait_upcasting` feature to the language.

### Motivation, use-cases, and solution sketches

* The `trait_upcasting` feature adds support for trait upcasting coercion. This allows a
trait object of type `dyn Bar` to be cast to a trait object of type `dyn Foo`
so long as `Bar: Foo`.

```rust,edition2018
#![feature(trait_upcasting)]

trait Foo {}

trait Bar: Foo {}

impl Foo for i32 {}

impl<T: Foo + ?Sized> Bar for T {}

let bar: &dyn Bar = &123;
let foo: &dyn Foo = bar;
```

### Non-goals

* To support `dyn Trait1 + Trait2` for general traits (this syntax naturally works, and should continue to work, with auto traits).

## Membership

| Role | Github |
| ---  | --- |
| [Owner] | [crlf0710](https://github.com/crlf0710) |
| [Liaison] | [nikomatsakis](https://github.com/nikomatsakis) |
