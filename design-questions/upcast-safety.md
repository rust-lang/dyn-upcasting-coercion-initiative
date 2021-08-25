# Upcast safety

## Scenario

* Casting `*dyn Foo` to `*dyn Bar` requires adjusting the vtable from a vtable for `Foo` to one for `Bar`
* For raw pointers, the metadata can be supplied by the user with `from_raw_parts`
    * If that metadata is incorrect, this could cause UB:
        * our current vtable format requires loads, and the pointers may not be valid
        * a flat vtable layout could give rise to out-of-bounds loads
* Unsafety is needed, but where?

## Options

### Unsafe to create a `*dyn Foo`

> Announce that every fat pointer needs to have valid metadata part. ~~Needs to switch the std::ptr::from_raw_parts{,_mut} APIs to be unsafe.~~ And updates other documentations.

**Implication:** Not able to create a "null pointer" version of `*dyn Foo` unless:

* You use `Option<*dyn Foo>`, of course
* We create some kind of "dummy" vtable that is structurally correct but has no actual functions within it; we would need a function for creating "valid-but-default" metadata as part of custom DST

Update:
  * It was pointed out by `steffahn` that `std::ptr::from_raw_parts` won't create fat pointer with invalid metadata.
  * One of the remaining ways to create a pointer with invalid metadata is [by using `transmute`](https://github.com/rust-lang/rust/issues/81513#issuecomment-798158332).

Question:
  * Does the language UB happen at `transmute` site or coercion site?

### Alter vtable layout to be flat

> Make vtables "flat", by removing all pointer indirections in vtables and appending all the data to the tail. This makes upcasting coercion codegen become adding an offset to the metadata scalar, so won't cause real UB. Will waste some more static bytes in multiple inheritance cases than before, might make embedded-dev people unhappy.

Question from Niko:
  * Is this sufficient? Does it imply we can't use `InBoundsGep`? Is that already true?

### Make raw pointer unsizing unsafe

> Announce that raw pointer unsizing coercion must happen in unsafe blocks, while other unsizing coercions can happen outside an unsafe block. This is actually a small breaking change. So need a future compat lint to migrate existing users dealing with raw pointers and some more changes to std(POC PR at #88239 explains the details but it's a bit long). A few other MIR-level details become observable by user: whether the compiler thinks it's a unsizing coercion or not.
>
> nikomatsakis: (cc @RalfJ, if you happen to be around)

## Conversation log

* [2021-08-25](https://zulip-archive.rust-lang.org/stream/144729-wg-traits/topic/object.20upcasting.html#250616592)