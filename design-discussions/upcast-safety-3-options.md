# Upcast safety options considered

This document details a number of options that were considered while preparing the [upcast-safety-3](./upcast-safety-3.md) proposal.

## Summary table

| proposal | unconditional-upcast | can-create-from-zeroed | can-create-from-arbitrary-bits | niche-for-dyn | can-release-something-zeroed-to-safe |
| --- | --- | --- | --- | --- | --- |
| fully-valid-vtable | :ballot_box_with_check:   | :octagonal_sign:  | :octagonal_sign: | :ballot_box_with_check: | :octagonal_sign: |
| null-placeholder | :octagonal_sign:  | :ballot_box_with_check:  | :octagonal_sign: | :ballot_box_with_check: | :octagonal_sign:
| operations-are-ub | :ballot_box_with_check:  | :ballot_box_with_check:  | :ballot_box_with_check: | :octagonal_sign: | :octagonal_sign: |
| operations-are-ub-nza | :ballot_box_with_check:  | :octagonal_sign:  | :octagonal_sign: | :ballot_box_with_check: | :octagonal_sign: |

## True no matter what

* `MaybeUnit<*const dyn Foo>` can be used to create an uninitialized `*const dyn Foo` easily, like with any other type.
* If we add `*const self` methods (which we should), and permit those to be invoked from safe code (which would be consistent, given that one can invoke a `self` method defined on a `*const T` type in safe code), then it will be *unsound* (i.e., potentially allow safe code to create UB) to release a `*const dyn Foo` to safe code unless the vtable is *fully valid* (created by the compiler).
* vtable layout is not defined.
* No matter what proposal we adopt, if you create a `*const dyn Foo` with anything less than a fully valid vtable (including e.g. a null placeholder), **you must not allow that to escape to arbitrary code**
    * This is actually a consequence of wanting to support `*const self` methods and method dispatch -- it could conceivably be true only for a trait `Foo` is it includes a `*const self` method, since those are not yet legal. 

## Proposals

### fully-valid-vtable

It is UB to create a `*const dyn Foo` unless the vtable is fully valid (given that vtable layouts are undefined, this means produced by the compiler for the time being).

If you need to create a `*const dyn Foo` and don't have a valid vtable available, you can use `MaybeUninit<*const dyn Foo>`, or a union, or `Option`, which is probably good practice.

Forwards compatibility:

* Given that vtable layouts are not defined, there isn't much you can do at present in unsafe code with a vtable besides invoking methods, upcasting, computing the size, or other such operations.
* If we were to adopt other proposals that weaken the validity requirement, compatibility is a bit complex. At the time we make the change, all `*const dyn` values in extant code would have valid vtables. But people would be able to write code that created values with less-than-valid vtables (e.g., maybe null, etc, depending on what change we made). They would not however be able to give that code to other functions, even unsafe functions, unless those functions stated explicitly that they don't require the `*const dyn` value to have a fully valid vtable (i.e., they promised not to upcast it, etc).
    * This is effectively the same as the operations-are-ub proposal, even if we adopt it today. You can create invalid vtables, but you cannot allow them to escape.

### null-placeholder

It is UB to create a `*const dyn Foo` unless the vtable is "sufficiently valid", except that NULL is permitted as a placeholder value.

This has the consequence that when we upcast, we have to check for NULL, which is less efficient.

Upshot:

* You may create a `*const dyn Foo` from `std::mem::zeroed`.
* The only thing you can do on a `*const dyn Foo` that is zeroed is copy it and upcast it.
    * *Currently*, this means you can release a `*const dyn Foo` to safe code, but if we were to add `*const self` methods, that might no longer be true, unless those methods are also prepared to deal with the null pointer in some way (e.g., abort?), or unless it's unsafe to invoke such a method (seems odd). But that's really an issue for `*const self` methods to deal with, since it's already true today.
* If receiving one from an unknown source, you can assume the vtable is either value for some type or NULL (of course, you can't assume anything about the *data* pointer, and vtable layouts are unknown, so it's not clear how useful that is).

Forwards compatibility:

* We can adopt any other proposal later, but we might lose the niche.

Notes:

* We can in theory support a niche with a value like 0x1 for the vtable? 

### operations-are-ub

There is no validity invariant for `*const dyn Foo` -- the vtable can be arbitrary. However, the only "non-UB" operation that you can do on a `*const dyn Foo` is to copy it, unless the vtable is known to be valid (produced by the compiler). Without a known valid vtable, all other options, including but not limited to the following, are UB:

* Upcasting it do `*const dyn Bar` (where `trait Foo: Bar`)
* Invoking methods from `Foo` (not currently possible without arbitrary-self-types)
* Creating a safe pointer type (e.g., `&*foo`)

Upshot:

* You can create a `*const dyn Bar` with `mem::zeroed`, but it must not escape to safe code (that would permit the safe code to create UB, and hence be unsound).
* If you are going to do anything besides copy that value around, it must be known to have a valid vtable at that time, or UB will result.

Forwards compatibility:

* If we adopt this proposal, we can't move to the others without introducing UB into otherwise valid code (consequence of the can-create-from-arbitrary-bits)

### operations-are-ub-nza

The validity invariant for `*const dyn Foo` is that the vtable must be non-zero and aligned, ensuring a niche. However, -- the vtable can be arbitrary. However, the only "non-UB" operation that you can do on a `*const dyn Foo` is to copy it, unless the vtable is known to be valid (produced by the compiler). Without a known valid vtable, all other options, including but not limited to the following, are UB:


## Properties

### unconditional-upcast

If this property holds, then...

```rust
trait Foo: Bar { }

let x: *const dyn Foo = ...;
let y: *const dyn Bar = x;
```

can be done without any conditional operations.

### can-create-from-zeroed

If this property holds, then...

```rust
let x: *const dyn Foo = std::mem::zeroed();
```

...is not insta-UB.

### can-create-from-arbitrary-bits

If this property holds, then...

```rust
let x: *const dyn Foo = /* arbitrary bits */;
```

...is not insta-UB.


### niche-for-dyn

Raw pointers for sized types do not presently have niches -- but under some of these proposals, vtables could potentially act as a niche, such that `Option<*const dyn Foo>` has the same size as `*const dyn Foo`. This seems like a pretty niche (no pun intended) advantage, and is not clearly desirable, but it helps to illustrate certain things.

Note that in some cases this property would be lost if we made future 'forwards compatible' changes (e.g., by moving from a more restricted variant to operations-are-ub).

## Removed proposals

There were some ideas that turned out to not add value and were removed from the vtable.

### sufficiently-valid-vtable

An earlier doc proposed a "sufficiently valid" vtable as the condition, where it was only legal to create a `*const dyn Foo` with a valid vtable, but you could not generally assume the vtable was valid. However, it was pointed out that because vtable layout is unstable, it's not clear what this "sufficiently valid" language adds. 

The original intent was to prevent unsafe code authors who receive a `*const dyn Foo` from assuming that the vtable was valid. The fear was that if we made changes that permitted a "less than fully valid" vtable, that unsafe code would now be getting fewer guaranteese than it had before. But given that vtable layouts are unstable, the code can't do much in practice, and in any case, if we *did* change the requirements, we could safe that "unsafe code which doesn't say otherwise requires a fully valid vtable".

Spelling out the scenarios:

1. when I release to safe code, full validity is required, because there are safe operations that require it
2. so if I write a safe function, I can always assume full validity on input
3. if I write an unsafe fn, in Rust *now* I can assume that I get full validity (because we defined that as the invariant), but the only thing I can do with that are defined operations at present (e.g., invoke methods, etc). If in the future we define the layout, I could read data from that vtable, but since it is required to be fully valid, that seems ok. In the future, we could loosen the validity requirement, but we would say that unsafe functions must explicitly state that they accept a `*const dyn` with a less than fully valid vtable (in other words, we'd have "edition-like" treatment of validity invariants.

Regardless, it's not clear what kind of "less than fully valid" vtables we would accept. Some possibilities:

* Null placeholder.
* A "skeleton" vtable that permits upcast but has invalid methods. Feels awfully niche, not worth it.

### 