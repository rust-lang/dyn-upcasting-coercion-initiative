# Dyn upcast Charter

## Goals

* To permit upcasting from `dyn Trait1` to `dyn Trait2` where `Trait1` is a subtype of `Trait2`

See the proposal in [rust-lang/lang-team#98](https://github.com/rust-lang/lang-team/issues/98) for more information.

## Non-goals

* To support `dyn Trait1 + Trait2` for general traits (this syntax naturally works, and should continue to work, with auto traits).

## Membership

**Owner:** Charles Lew [(crlf0710)](https://github.com/crlf0710)
**Team Liason:** Niko Matsakis [(nikomatsakis)](https://github.com/nikomatsakis)
