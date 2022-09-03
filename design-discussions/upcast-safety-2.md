# Dyn upcast safety

**NB: This document is outdated.** [Click here to see the latest version.](./upcast-safety-3.md)

## Summary

This doc concerns possible resolutions for the following potential problem:

* Given:
    * unsize coercions of raw pointers are legal and stable in safe code
    * upcasting may require loading data from vtable
* If:
    * dyn upcast coercions are a kind of unsize coercion
    * the [safety invariant][vvsi] for `*const dyn Foo` permits an invalid vtable
* Then:
    * safe code could cause UB by upcasting a `*const dyn SubTrait` to a `*const dyn SuperTrait`

The doc recommends:

* Defining the safety invariant for a `*const dyn Foo` to be a "sufficiently valid" vtable for `Foo`
    * the definition of "sufficiently valid" is intentionally left undefined except to say:
        * a fully valid vtable is also "sufficiently valid"
        * it permits upcasting from `*const dyn SubTrait` to `*const dyn SuperTrait` without UB
    * this implies that the only way for users to produce a "sufficiently valid" vtable is with a fully valid one, but conversely safe code cannot rely on a `*const dyn Foo` having a *fully* valid vtable

## Background

### Unsize coercions are possible in safe code

A brief review of the [DST coercion design](https://github.com/rust-lang/rfcs/blob/master/text/0982-dst-coercion.md):

* The `PT: CoerceUnsized<PU>` trait indicates that a smart pointer `PT` can be coerced to a smart pointer `PU`
    * This trait is manually implemented for each smart pointer type, e.g. `impl<T> CoerceUnsized<Rc<U>> for Rc<T> where T: Unsize<U>`.
    * The compiler requires that `Rc<T> -> Rc<U>` be a legal DST coercion per various built-in rules (e.g., `T` must be stored in the final field of the struct and cannot be behind another pointer indirection).
    * Example: `Rc<String>: CoerceUnsized<Rc<dyn Debug>>`
* The `T: Unsize<U>` trait indicates that the `T: ?Sized` referent can be "unsized" to the reference `U`
    * This trait is automatically implemented by the compiler.
    * Example: `String: Unsize<dyn Debug>`
    * Example: `[String; 32]: Unsize<[String]>`

This design permits one to write generic code that performances unsized coercions, but the trait is unstable so this only works on nightly:

```rust
// Unstable:
fn foo<PT, PU>(pt: PT) -> PU 
where 
    PT: CoerceUnsized<PU>,
{
    pt
}
```

[Example](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=0b3aef50e32fb224bc053f7ef4640e94)

You can however coerce from a `*const T` to a `*const dyn Trait` in safe code on stable ([playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=695eeb19aa21e4762b9f5025fea7176c
)):

```rust
fn main() {
    let x: *const u32 = &32;
    let y: *const dyn std::fmt::Debug = x;
}
```

### Upcasting `dyn SubTrait` to `dyn SuperTrait` is considered an unsize coercion

We implemented upcasting from one dyn trait to a supertrait as an upsizing coercion. This means that `dyn SubTrait: Unsize<dyn SuperTrait>`. This is a coercion because it requires adjusting the vtable.

### How upcasting coercions adjust the vtable

The vtable for a `dyn SubTrait` [now embeds pointers](https://rust-lang.github.io/dyn-upcasting-coercion-initiative/design-discussions/vtable-layout.html) to the vtables for super traits. Upcasting therefore requires loading the new vtable for the supertrait from the specific slot within the subtrait's table.

There are alternatives we could use such that upcasting would be a pure integer adjustment with no load, but that would be less efficient in terms of space usage.

Although not directly relevant here, there is another operation that requires accessing the vtable, which is finding the offset of fields -- see [the discussion in the appendices](#mfo) for details. This operation is however only permitted in unsafe code because it requires a dereference of the raw pointer.

### Ergo, upcasting raw pointers is possible in safe code

Without any further changes, the following upcasting is legal in safe Rust ([playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=37e49f69ebf992856b07a430c4d5510f)):

```rust
#![feature(trait_upcasting])]

trait Sub: Sup { }
trait Sup { }

fn convert(x: *const dyn Sub) -> *const dyn Sup {
    x
}
```


### Raw pointers are traditionally permitted to be "garbage"

A sized raw pointer like `*const u32` has no validity/safety invariants to speak of. It is not required to be aligned, it may be null, and it may point at arbitrary memory. This is why `Option<*const u32>` requires an extra word fo the discriminant and it is why dereferencing a raw pointer is unsafe.

However, as noted in the previous section:

* if upcasting raw pointers is possible in safe code and
* if upcasting requires loading data from the vtable

then the safety condition of `*const dyn Foo` must include a "sufficiently valid" vtable. Sufficiently valid means that it is "structurally complete" in that it contains valid pointers that can be loaded to do upcasting.

### What do we do for null or garbage data?

The challenge with `*const dyn Trait` is "what do we do to make a null pointer"? For better or worse, `*const T` pointers can traditionally be null: but when one creates a vtable, one is supposed to have some underlying data type to make the vtable for, and with a null pointer that is not possible.

### Creating a `*const dyn`

If we wish to ensure a safety invariant for `*const dyn` values, we have to ask ourselves how one could go about producing such a value. There are currently two ways to create a `*const dyn Trait`, one safe and one unsafe:

* [`std::ptr::from_raw_parts`](https://doc.rust-lang.org/std/ptr/fn.from_raw_parts.html) -- safe, unstable
* [`std::mem::transmute`](https://doc.rust-lang.org/std/mem/fn.transmute.html) -- unsafe, unwise

There is also a third way to make a `*const T` that doesn't work for `dyn` now but which could be made to work:

* [`std::ptr::null`](https://doc.rust-lang.org/std/ptr/fn.null.html) -- safe, stable, but currently limited only to `*const T` where `T: Sized`

Of these three:

* `from_raw_parts` requires a valid vtable for `T` as argument, so it would meet the safety requirement
* `transmute` is unsafe, but it would indeed be a requirement that users of transmute must uphold
* `null`, if extended to unsized types, would be tricky -- we would need to have some way to get a "dummy" vtable that is structurally sound enough to permit upcasting, but which has (for example) null pointers for the actual methods. This is, however, eminently doable.

### Interaction: Raw pointer method calls

It would be useful if unsafe code could declare `*const self` and `*mut self` methods in traits

```rust
pub trait FreeMe {
    pub unsafe fn free_me(*const self);
}
```

Given that using `&self` implies (for ex

## Core decision to be made

The core decision to be made is to choose between two paths:

* Splitting out "unsafe" unsizing operations from safe ones;
    * Unsizing operations on safe pointers like `&dyn SubTrait` would continue to work as they do today.
    * Implicit unsizing operations on raw pointers like `*const dyn SubTrait` would work sometimes, but those unsizing operations that require the metadata meet some sort of condition would require an explicit function call (e.g. the proposal below adds an unsafe function `unsize_raw_pointer`).
* Raw pointers have a validity or safety invariant that puts conditions on metadata 
    * There are options in terms of when this invariant must hold and how strict the invariant must be, but the upshot is that whenever you synthesize the metadata for a raw, wide pointer (e.g., from a transmute), you need to ensure that this metadata comes from some valid source, and is not just garbage. This is contrast to the data pointer, which can generally be arbitrary bytes (though not unininitialized).

### Why prefer unsafe raw pointer upcasting?

Raw pointers have traditionally been viewed as "untrusted data until they are used". This is why dereferencing a raw pointer is unsafe: that's the point where it must be valid. It makes sense to extend this model to the metadata as well. When working with raw pointers and unsafe code, implicit operations like upcasting are a bug, not a feature, so it's useful to segregate them out and make them explicit via some kind of function call. It does require adding a new trait (see below) to the unsizing mechanism, but it only requires an internal "implementation" trait, and doesn't affect the "main trait" (`CoerceUnsized`).

Besides, so long as the validity/safety invariants remain in flux (which they will be for a while yet), this choice is forwards compatible with the others. We have the option to remove the "unsafe upcast" and merge it with safe upcast and strengthen the relevant invariant(s).

### Why prefer unsafe some form of invariant?

It's not clear why one would ever have invalid metadata in a wide pointer to start with; it's not an easy thing to do, you have to basically transmute from random bytes (e.g., zeroed memory). If you want to have a garbage pointer that is not yet initialized, use `MaybeUninit`. If you want a null pointer, use `ptr::null` or `Option<Unique<_>>`. In exchange for following these best practices, you get two things:

* Safe unsafe code overall, since you are being clearer about your intentions.
* A simpler coercion model, with fewer traits and moving parts, and things that work the same for all kinds of pointers
* A language that is 

## Options

The following options have been identified. The preferred solution is not yet clear.

### RawUnsafe: Make raw pointer upcasting unsafe (not possible once `dyn SubTrait: Unsize<dyn SuperTrait>` is stable)

We could permit "safe pointer" upcasting but make raw pointer upcasting unsafe. This would require changing the design of the coercion traits somewhat. We would introduce a new "magic trait" `RawUnsize`, analogous to `Unsize` except that it doesn't permit upcasting or any other operations that could require valid metadata. We would then modify the impl of `CoerceUnsized` for raw pointers to be:

```rust
impl<T, U> CoerceUnsized<*const U> for *const T where
    T: Unsize<U> + ?Sized,
    U: ?Sized, 
```

and it would become


```rust
impl<T, U> CoerceUnsized<*const U> for *const T where
    T: RawUnsize<U> + ?Sized,
    U: ?Sized, 
```

To support upcasting on raw pointers, we can then introduce some other intrinsic for doing raw pointer upcast, such as something like this (modulo stacked borrows, which this *particular* definition may invalidate):

```rust
/// Unsafefy condition: If the metadata for `T` must be valid.
pub unsafe fn unsize_raw_pointer<T, U>(t: *const T) -> *const U
where
    T: Unsize<U>,
{
    let (_, t_metadata) = t.into_raw_parts();
    unsafe { &*t }
}
```

So long as `Unsize` remains a strict superset of `RawUnsize`, we could change things in the future to make `RawUnsize` an alias for `Unsize` (or, if it is unstable, remove it altogether) and thus deprecate the `unsize_raw_pointer` function. This is therefore forwards compatible with the preferred proposal here as well as other things that say "still possible in the future".
### VISufficientlyValid: Extend the validity invariant of `dyn Trait` to require a "sufficiently valid" vtable

One solution is to extend the [validity invariant][vvsi] for raw pointers to require a "sufficiently valid" vtable. We don't specify the precise condition that makes a vtable "sufficiently valid" except to say that a fully valid vtable is "sufficiently valid", and that a "sufficiently valid" vtable permits dyn upcasting without UB.

Implications:

* Whenever one creates a wide pointer, one must ensure that the metadata is "sufficiently valid":
    * For `*const dyn Trait`, this would mean that one must use a valid vtable for `Trait`.
        * In particular, we don't define what is "sufficiently valid" so you have to use something that is fully valid; at the same time, you cannot rely on the vtable for `*const dyn Trait` being fully valid yourself, only "sufficiently valid" (which is "valid enough for upcast" and that's it).
* If the pointer is not initialized, and hence you don't know which vtable to use, you have the following options:
    * Use a dummy vtable for any type, it doesn't matter which.
    * Use `MaybeUninit<*const dyn Foo>`, in which case no safety invariant is assumed.
    * Use `Option<*const dyn Foo>` and `None` instead of null: safer, wastes space.
    * Use `Option<NonNull<dyn Foo>>` and `None` instead of null: safer, generally better, perhaps less ergonomic.

One downside of this proposal is that the validity invariant is stricter than is needed: that is, the purpose of the validity invariant is primarily to enable the compiler's ability to perform layout optimizations. This rule would enable the compiler to silently insert upcasting operations if it needed to do so, but it's not clear why it would need to do that spontaneously: those operations are always tied to something else (e.g., a coercion or a method call). Therefore, the safety invariant might seem like a better fit.

### SISufficientlyValid: Extend the safety invariant of `dyn Trait` to require a "sufficiently valid" vtable

As an alternative to modifying the validity invariant, we could modify the [safety invariant][vvsi] for wide pointers to include "sufficiently valid" metadata (see VISufficientlyValid for details). 


Implications:

* Whenever one performs an upcast or other operation with a wide pointer, one must ensure that the metadata is "sufficiently valid":
* If the pointer is not initialized, and hence you don't know which vtable to use, you have the same options as described under VISufficientlyValid.

The primary downsice of this proposal versus VISufficientlyValid is that the causes of UB are rather more subtle. Instead of UB occurring when the pointer is created, it occurs when an upcast occurs, and the locations for upcasts can be implicit (eg., any assignment or function call). Consider the following example:

```rust
fn noop(x: *mut dyn SuperTrait) { 
    /* look ma, empty body */
}

fn creator() {
    // No UB yet: the metadata for `x` is not sufficiently valid,
    // but we haven't done anything with it yet.
    let x: *mut dyn SubTrait = unsafe { std::mem::zeroed() };

    // `y = x` does not trigger UB, just a copy.
    let y = x;

    // UB! Here there is a coercion.
    noop(y);
}
```

For this reason, it would probably be "best practice" to treat this condition "as if" it were part of the validity invariant.

### SIFullyValid: Extend safety condition to require a "fully valid" vtable (still possible in the future)

This would permit safe code to read values from the vtable of a `*const dyn Trait` without any unsafety (just as it does for upcasting). It's not clear why we would want to permit this, and it may foreclose useful options in the future.

Adopting this option remains a possibility in the future, as it would be a backwards compatible extension to the above rule.

### SIStructValid: Extend safety condition to require a "structurally valid" vtable (still possible in the future)

Instead of requiring a valid vtable, we could require a "structurally valid" vtable. This vtable would have null pointers for all methods as well as a dummy type-id but would have the same "structure" as an ordinary vtable. There would be an intrinsic that gives access to the vtable:

```rust
fn dummy_vtable::<T: ?Sized>() -> T::Metadata
```

This would permit one to represent an uninitialized dyn pointer as `*const dyn Foo` and use the dummy-vtable for its metadata. This could be convenient, but is less typesafe than `MaybeUnit` or `Option<NonNull>` and not obviously better.

Adopting this option remains a possibility in the future, as it would be a backwards compatible extension to the above rule.

### NullVtable: Special case the vtable for null (still possible in the future)

Instead of saying that a "null dyn pointer" must have a structurally sound vtable, we could permit null as the value for the vtable. This would require an "if branch" or some kind of more complex logic in the upcasting path, since we couldn't unconditionally do a load. That might be acceptable, but it seems silly to slow down the upcasting path for a relatively unusual case of having a null 

### FlatVtable: Adjust vtable layout to not require a load (still possible in the future)

We could adjust the vtable layout for a subtrait to include embedded copies of all supertraits. This way the upcast is a pure offset operation and does not require a load. This would be less efficient in terms of space usage. We generally prefer not to limit the possible vtable designs that an implementation can use unless we have to, so as to leave room for future developments.

## Appendices

[vvsi]: #validity-versus-safety-invariants

### Validity versus safety invariants

Let us take a digression to cover Ralf's distinction of [validity vs safety invariants](https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html):

* The **validity invariant** for a type `T` defines the invariant that **all values of type T must meet, all of the time**. These invariants are respected in both safe and unsafe code and are primarily used to do layout optimizations. 
    * For example, the validity invariant of `bool` requires that the value is 0 or 1 but not 2. Thanks to this validity invariant, `Option<bool>` can be represented by the compiler in the same amount of space as `bool`.
* The **safety invariant** for a type `T` defines the invariant that **all values of type `T` must meet when given to safe code**. These invariants are used to justify unsafe code, but aren't understood by the compiler.
    * For example, a vector has a field for its length and capacity, and the safety invariant requires that they accurately describe the allocate space available in the vector's buffer (and aren't just random values). Thanks to this safety invariant, we can create `push` as a safe function: it can read those fields and rely on them being accurate to decide whether the memory still has any free capacity.

[mfo]: #metadata-and-field-offsets

### Metadata and field offsets

Consider these struct definitions:

```rust
struct RefCounted<T> {
    counter: usize,
    data: T,
}

struct Foo<U> {
    field: u32,
    data: U
}
```

and now assume that I have a `*const RefCounted<Foo<u32>>` which is coerced:

```rust
let pointer1: *const RefCounted<Foo<u32>> = ...;
let pointer2: *const RefCounted<Foo<dyn Debug>> = pointer1;
```

If I now try to get the address of the field `field`...

```rust
let pointer3: *const u32 = unsafe { raw_addr!((*pointer2).data.field) };
```

...this operation requires valid metadata. This is because the offset of `field` is a function of the alignment of `Foo<U>`, which is a function of the alignment of `U`. In these cases, the compiler will load the alignment from the vtable and do the required address computations itself.