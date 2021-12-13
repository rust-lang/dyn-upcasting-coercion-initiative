# Dyn upcast safety

## Summary

This doc concerns possible resolutions for the following potential problem:

* Given:
    * unsize coercions of raw pointers are legal and stable in safe code
    * upcasting may require loading data from vtable
* If:
    * dyn upcast coercions are a kind of unsize coercion
    * the [safety invariant][appendix] for `*const dyn Foo` permits an invalid vtable
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

(Although not directly relevant here, there is another option that requires accessing the vtable from safe code, which is finding the offset of fields. This is because the alignment of a `*const dyn Foo` depends on the hidden type and hence we must load the alignment from the vtable and do offsetting adjustments at runtime.)

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

## Options

### Preferred: Extend the safety invariant of `dyn Trait` to require a "sufficiently valid" vtable

Our preferred solution is to extend the safety invariant for raw pointers to require a "sufficiently valid" vtable. We don't specify the precise condition that makes a vtable "sufficiently valid" except to say that a fully valid vtable is "sufficiently valid", and that a "sufficiently valid" vtable permits dyn upcasting without UB.

Implications:

* The only way to meet the safety invariant for a `*const dyn Trait` is to use a valid vtable for `Trait`.
    * In particular, we don't define what is "sufficiently valid" so you have to use something that is fully valid; at the same time, you cannot rely on the vtable for `*const dyn Trait` being fully valid yourself, only "sufficiently valid" (which is "valid enough for upcast" and that's it).
* If the pointer is not initialized, and hence you don't know which vtable to use, you have the following options:
    * Use a dummy vtable for any type, it doesn't matter which.
    * Use `MaybeUninit<*const dyn Foo>`, in which case no safety invariant is assumed.
    * Use `Option<*const dyn Foo>` and `None` instead of null: safer, wastes space.
    * Use `Option<NonNull<dyn Foo>>` and `None` instead of null: safer, generally better, perhaps less ergonomic.

## Other options

The following options were rejected. This section briefly explains why.

### Extend safety condition to require a "fully valid" vtable (still possible in the future)

This would permit safe code to read values from the vtable of a `*const dyn Trait` without any unsafety (just as it does for upcasting). It's not clear why we would want to permit this, and it may foreclose useful options in the future.

Adopting this option remains a possibility in the future, as it would be a backwards compatible extension to the above rule.


### Extend safety condition to require a "structurally valid" vtable (still possible in the future)

Instead of requiring a valid vtable, we could require a "structurally valid" vtable. This vtable would have null pointers for all methods as well as a dummy type-id but would have the same "structure" as an ordinary vtable. There would be an intrinsic that gives access to the vtable:

```rust
fn dummy_vtable::<T: ?Sized>() -> T::Metadata
```

This would permit one to represent an uninitialized dyn pointer as `*const dyn Foo` and use the dummy-vtable for its metadata. This could be convenient, but is less typesafe than `MaybeUnit` or `Option<NonNull>` and not obviously better.

Adopting this option remains a possibility in the future, as it would be a backwards compatible extension to the above rule.

### Special case the vtable for null (still possible in the future)

Instead of saying that a "null dyn pointer" must have a structurally sound vtable, we could permit null as the value for the vtable. This would require an "if branch" or some kind of more complex logic in the upcasting path, since we couldn't unconditionally do a load. That might be acceptable, but it seems silly to slow down the upcasting path for a relatively unusual case of having a null 

### Make raw pointer upcasting unsafe (not possible once `dyn SubTrait: Unsize<dyn SuperTrait>` is stable)

Raw pointer unsizing coercions are currently stable and legal in safe code, as demonstrated above, so we cannot simply make all such coercions illegal. What we could do is have a distinct *set* of coercions, perhaps a `T: UnsafeUnsize`, such as that coercion is legal if `T: UnsafeUnsize` is implemented and we are in an unsafe block. This complicates the coercion mechanism which is already fairly complex.

### Adjust vtable layout to not require a load (still possible in the future)

We could adjust the vtable layout for a subtrait to include embedded copies of all supertraits. This way the upcast is a pure offset operation and does not require a load. This would be less efficient in terms of space usage. We generally prefer not to limit the possible vtable designs that an implementation can use unless we have to, so as to leave room for future developments.

## Appendices

[appendix]: #validity-versus-safety-invariants

### Validity versus safety invariants

Let us take a digression to cover Ralf's distinction of [validity vs safety invariants](https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html):

* The **validity invariant** for a type `T` defines the invariant that **all values of type T must meet, all of the time**. These invariants are respected in both safe and unsafe code and are primarily used to do layout optimizations. 
    * For example, the validity invariant of `bool` requires that the value is 0 or 1 but not 2. Thanks to this validity invariant, `Option<bool>` can be represented by the compiler in the same amount of space as `bool`.
* The **safety invariant** for a type `T` defines the invariant that **all values of type `T` must meet when given to safe code**. These invariants are used to justify unsafe code, but aren't understood by the compiler.
    * For example, a vector has a field for its length and capacity, and the safety invariant requires that they accurately describe the allocate space available in the vector's buffer (and aren't just random values). Thanks to this safety invariant, we can create `push` as a safe function: it can read those fields and rely on them being accurate to decide whether the memory still has any free capacity.
