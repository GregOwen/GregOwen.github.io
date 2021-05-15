---
title: "How to install the latest nightly Rust that supports rustfmt (or any other component)"
date: 2021-05-15T12:40:00-07:00
editors:
  - Bianca Homberg
classes: wide
categories:
  - programming
tags:
  - rust
  - tools
---

**tl;dr:** `rustup toolchain install nightly --allow-downgrade -c rustfmt`

I recently had to debug a Rust macro in one of my personal projects. I wanted to use the excellent [cargo expand](https://github.com/dtolnay/cargo-expand) command to dump the expanded macro code, but ran into a dependency issue: `cargo expand` [can only run on nightly Rust](https://github.com/dtolnay/cargo-expand) and the version of `nightly` that I had installed didn't support `rustfmt` (which is required for `cargo expand` to pretty-print the expanded code)[^nightly-expectations].

The creators of `cargo expand` anticipated this exact situation, so they allow running the command in a mode that doesn't depend on `rustfmt`: `cargo expand --ugly` will skip the pretty-printing, so it should run on any version of `nightly` that supports `cargo expand`[^nightly-feature-flags]. But what happens if you really want your expanded macro code to be pretty-printed? Or, as was the case for my project, what happens if your *project itself* depends on `rustfmt`?

What I really wanted was to install the latest version of `nightly` that supported `rustfmt`. I didn't particularly care if I was a few days behind the most recent `nightly`, since I wasn't using any `nightly` features in my project itself. Fortunately, [rustup](https://rustup.rs/) supports exactly this use case:

```
rustup toolchain install nightly --allow-downgrade -c rustfmt
```

The `-c rustfmt` option tells `rustup` that we want to download the `rustfmt` component along with the rest of the `nightly` toolchain. The `--allow-downgrade` option tells `rustup` to install the latest version of `nightly` that supports all of the components we asked for[^allow-downgrade-pr].

My initial thought had been that this would be managed via `rustup component add` (my mental model was that my goal was to add the `rustfmt` component to `nightly`, rather than downgrade `nightly` until it supported `rustfmt`). It turns out that there was [an attempt](https://github.com/rust-lang/rustup/pull/2176) to add support for `--allow-downgrade` to `rustup component add` but it turned out to be trickier than expected (there's an [open issue](https://github.com/rust-lang/rustup/issues/2146) if you want to help out). Fortunately, the `rustup toolchain install` approach does what we need.

[^nightly-expectations]: The fact that `rustfmt` (or any given [component](https://rust-lang.github.io/rustup/concepts/components.html)) wasn't available on the version of `nightly` that I was running isn't particularly surprising. `nightly` updates frequently and doesn't have the same stability guarantees as, well, `stable`, so even heavily-used components like `rustfmt` will often not work on some versions of `nightly`. From what I can tell, `rustfmt` tends to catch up within a few days so it's usually possible to find a `nightly` version that supports `rustfmt` and isn't too far out of date.

[^nightly-feature-flags]: As a side note, this is a really great user experience from a cargo command that must be run on `nightly`. If you know that your command can only run on `nightly` and that it depends on components that may not be present on any given version of `nightly`, it's really nice to provide users a way to toggle off those dependencies.

[^allow-downgrade-pr]: Here's [the PR](https://github.com/rust-lang/rustup/pull/2126) that added --allow-downgrade if you want to see the discussion
