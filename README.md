# Dyn upcasting coercion initiative

![initiative status: active](https://img.shields.io/badge/status-active-brightgreen.svg)

## What is this?

This page tracks the work of the dyn upcasting coercion [initiative]! To learn more about what we are trying to do, and to find out the people who are doing it, take a look at the [charter]. 

[charter]: ./CHARTER.md
[initiative]: https://lang-team.rust-lang.org/initiatives.html

## Current status

The following table lists of the stages of an initiative, along with links to the artifacts that will be produced during that stage.

| Stage                                 | State | Artifact(s) |
| ------------------------------------- | ----- | ----------- |
| [Proposal]                            | ✅    | [Proposal issue](https://github.com/rust-lang/lang-team/issues/98) |
|                                       |       | [Charter](./CHARTER.md) |
|                                       |       | [Tracking issue](https://github.com/rust-lang/rust/issues/65991) |
| [Development]                         | 🦀    | [Explainer](./explainer.md) | 
| [Feature complete]                    | 💤    | Stabilization report |
| [Stabilized]                          | 💤    | |

[Proposal]: https://lang-team.rust-lang.org/initiatives/process/stages/proposal.html
[Experimental]: https://lang-team.rust-lang.org/initiatives/process/stages/proposal.html
[Development]: https://lang-team.rust-lang.org/initiatives/process/stages/development.html
[Feature complete]: https://lang-team.rust-lang.org/initiatives/process/stages/feature-complete.html
[Stabilized]: https://lang-team.rust-lang.org/initiatives/process/stages/stabilized.html

Key:

* ✅ -- phase complete
* 🦀 -- phase in progress
* 💤 -- phase not started yet

## How Can I Get Involved?

* Check for 'help wanted' issues on this repository!
* If you would like to help with development, please contact the [owner](./charter.md#membership) to find out if there are things that need doing.
* If you would like to help with the design, check the list of active [design discussions](./design-discussions/README.md) first. 
* If you have questions about the design, you can file an issue, but be sure to check the [FAQ](./FAQ.md) or the [design-discussions](./design-discussions/README.md) first to see if there is already something that covers your topic.
* If you are using the feature and would like to provide feedback about your experiences, please [open a "experience report" issue].
* If you are using the feature and would like to report a bug, please open a regular issue.

We also participate on [Zulip][chat-link], feel free to introduce yourself over there and ask us any questions you have.

[open issues]: /issues
[chat-link]: https://rust-lang.zulipchat.com/#narrow/stream/144729-wg-traits
[team-toml]: https://github.com/rust-lang/team/blob/master/teams/initiative-dyn-upcasting-coercion.toml

## Building Documentation
This repository is also an mdbook project. You can view and build it using the
following command.

```
mdbook serve
```
