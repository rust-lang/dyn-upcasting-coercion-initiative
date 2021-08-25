# What qualitifies as a dyn upcasting coercion

The condition comes from the existing infrastructure within the compiler.

Currently the unsizing coercion on a trait object type allows it to:

1. Removing one or more auto traits. (i.e. `dyn Foo + Send: Unsize<dyn Foo>`)

2. Changing lifetime bounds according to subtyping rules. (`dyn Bar<'static>: Unsize<dyn Bar<'a>>`)

3. Changing the principal trait to one of its supertraits. (`dyn Goo: Unsize<dyn Foo>` where `Goo` is `trait Goo: Foo {}`)

When the third rule is involved, this unsizing coercion is a dyn upcasting coercion.


