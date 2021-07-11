---
title: "Understanding #[derive(Clone)] in Rust"
date: 2021-06-08T19:54:00-07:00
classes: wide
categories:
  - programming
tags:
  - rust
---

# A Rust riddle

Let's start with a Rust riddle. This code compiles:

```rust
use std::rc::Rc;

trait NotNecessarilyClone {}

#[derive(Clone)]
struct CloneByRc<NNC> where NNC: NotNecessarilyClone {
  nnc: Rc<NNC>,
}
```

This on its own is not particularly surprising. We have a trait `NotNecessarilyClone`. Things that implement this trait might or might not also implement `Clone`[^clone-is-an-example]. We have a struct `CloneByRc` that stores an object that implements `NotNecessarilyClone`, but it stores it behind an [`Rc`](https://doc.rust-lang.org/std/rc/struct.Rc.html) so that `CloneByRc` can implement `Clone`. This is kind of the point of an `Rc`: `Rc<T>` [implements `Clone`](https://doc.rust-lang.org/std/rc/struct.Rc.html#impl-Clone) for all `T`. It takes something that isn't normally safe to clone and hides it behind a reference counter so that cloning just means incrementing the reference counter.

The surprising part is this: `CloneByRc<NNC>` doesn't actually implement `Clone`, even though we've used `#[derive(Clone)]`! If we try to use `CloneByRc<NNC>` somewhere that requires it to implement `Clone`, we get a compilation error:

```rust
trait MustBeClone: Clone {}

// Compilation error!
impl<NNC> MustBeClone for CloneByRc<NNC> where NNC: NotNecessarilyClone {}
```

Specifically, we get the error

```
error[E0277]: the trait bound `NNC: Clone` is not satisfied
  --> src/lib.rs:12:11
   |
10 | trait MustBeClone: Clone {}
   |                    ----- required by this bound in `MustBeClone`
11 |
12 | impl<NNC> MustBeClone for CloneByRc<NNC> where NNC: NotNecessarilyClone {}
   |           ^^^^^^^^^^^ the trait `Clone` is not implemented for `NNC`
   |
   = note: required because of the requirements on the impl of `Clone` for `CloneByRc<NNC>`
help: consider further restricting this bound
   |
12 | impl<NNC> MustBeClone for CloneByRc<NNC> where NNC: NotNecessarilyClone + Clone {}
   |                                                                          ^^^^^^^
```

At first, I was very confused about why this failed to compile. Why does Rust complain that `the trait Clone is not implemented for NNC`? I only want `CloneByRc<NNC>` to implement `Clone`. The whole point of using an `Rc` was so that `NNC` wouldn't *need* to implement `Clone`.

After that, I was very confused about why the original code compiled at all. If `CloneByRc<NCC>` doesn't actually implement `Clone`, why didn't the original code throw an error when I tried to `#[derive(Clone)]`?

Eventually, I was no longer confused about either of those things, which is why I'm writing this blog post.

# What `#[derive(Clone)]` promises

Let's start with what `#[derive(Clone)]` [claims it does](https://doc.rust-lang.org/std/clone/trait.Clone.html#derivable):

> This trait can be used with `#[derive]` if all fields are `Clone`. The derived implementation of `Clone` calls `clone` on each field.

Ok, that's pretty straightforward. If every field in a struct implements `Clone`, then you can just call `clone` on each field and now you've cloned the whole struct.

So why doesn't this work for my `CloneByRc<NNC>` struct? It only has one field, and that field implements `Clone`. The derived implementation should just be able to call `ncc.clone()` and be done.

To understand what's going on, we need to look at what happens when generics (in this case, the `<NNC>` type parameter) enter the picture.

# Implementing `#[derive(Clone)]`: a naive approach

Let's consider how we'd implement `#[derive(Clone)]` for a struct. We want to write a deriver function that will take in a description of the underlying struct, then emit Rust code that defines the implementation of `Clone` for that struct[^procedural-macros].

To keep things simple, let's just look at the Rust code that we're going to emit. "The derived implementation of `Clone` calls `clone` on each field" sounds pretty straightforward. We can imagine our implementation looking like this:

<table class="two-column-code">
  <tr>
    <td>
      <div>Struct</div>
    </td>
    <td>
      <div>Implementation</div>
    </td>
  </tr>
  <tr>
    <td>
    <div class="language-rust highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">struct</span> <span class="n">Foo</span> <span class="p">{</span>
  <span class="n">a</span><span class="p">:</span> <span class="nb">u32</span><span class="p">,</span>
  <span class="n">b</span><span class="p">:</span> <span class="nb">Vec</span><span class="o">&lt;</span><span class="nb">u32</span><span class="o">&gt;</span><span class="p">,</span>
  <span class="n">c</span><span class="p">:</span> <span class="nb">bool</span>
<span class="p">}</span>




</code></pre></div></div>
    </td>
    <td>
    <div class="language-rust highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">impl</span> <span class="n">Clone</span> <span class="k">for</span> <span class="n">Foo</span> <span class="p">{</span>
  <span class="k">fn</span> <span class="nf">clone</span><span class="p">(</span><span class="o">&amp;</span><span class="k">self</span><span class="p">)</span> <span class="k">-&gt;</span> <span class="n">Self</span> <span class="p">{</span>
    <span class="n">Foo</span> <span class="p">{</span>
      <span class="n">a</span><span class="p">:</span> <span class="k">self</span><span class="py">.a</span><span class="nf">.clone</span><span class="p">(),</span>
      <span class="n">b</span><span class="p">:</span> <span class="k">self</span><span class="py">.b</span><span class="nf">.clone</span><span class="p">(),</span>
      <span class="n">c</span><span class="p">:</span> <span class="k">self</span><span class="py">.c</span><span class="nf">.clone</span><span class="p">(),</span>
    <span class="p">}</span>
  <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>
    </td>
  </tr>
</table>

Once our macro emits this Rust code, the compiler will come along and check whether the code we emitted actually compiles. In this case, the code just calls `clone` on each field of the input struct. If every field implements `Clone`, then this will typecheck and the compiler will be happy. If some field doesn't implement `Clone`, the compiler will see us calling `field.clone()` on that field and throw an error.

This approach works fine as long as the input struct doesn't have any type parameters. But what happens if it does?

<table class="two-column-code">
  <tr>
    <td>
      <div>Struct</div>
    </td>
    <td>
      <div>Implementation</div>
    </td>
  </tr>
  <tr>
    <td>
<div class="language-rust highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">struct</span> <span class="n">Foo</span><span class="o">&lt;</span><span class="n">B</span><span class="p">,</span> <span class="n">C</span><span class="o">&gt;</span> <span class="p">{</span>
  <span class="n">a</span><span class="p">:</span> <span class="nb">u32</span><span class="p">,</span>
  <span class="n">b</span><span class="p">:</span> <span class="nb">Rc</span><span class="o">&lt;</span><span class="n">B</span><span class="o">&gt;</span><span class="p">,</span>
  <span class="n">c</span><span class="p">:</span> <span class="n">C</span><span class="p">,</span>
<span class="p">}</span>




</code></pre></div></div>
    </td>
    <td>
<div class="language-rust highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">impl</span><span class="o">&lt;</span><span class="n">B</span><span class="p">,</span> <span class="n">C</span><span class="o">&gt;</span> <span class="n">Clone</span> <span class="k">for</span> <span class="n">Foo</span><span class="o">&lt;</span><span class="n">B</span><span class="p">,</span> <span class="n">C</span><span class="o">&gt;</span> <span class="p">{</span>
  <span class="k">fn</span> <span class="nf">clone</span><span class="p">(</span><span class="o">&amp;</span><span class="k">self</span><span class="p">)</span> <span class="k">-&gt;</span> <span class="n">Self</span> <span class="p">{</span>
    <span class="n">Foo</span> <span class="p">{</span>
      <span class="n">a</span><span class="p">:</span> <span class="k">self</span><span class="py">.a</span><span class="nf">.clone</span><span class="p">(),</span>
      <span class="n">b</span><span class="p">:</span> <span class="k">self</span><span class="py">.b</span><span class="nf">.clone</span><span class="p">(),</span>
      <span class="n">c</span><span class="p">:</span> <span class="k">self</span><span class="py">.c</span><span class="nf">.clone</span><span class="p">(),</span> <span class="c">// problem!</span>
    <span class="p">}</span>
  <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>
    </td>
  </tr>
</table>

As before, the compiler will check our emitted code. `self.a.clone()` is fine, since `u32` implements `Clone`. `self.b.clone()` is fine, since `Rc<B>` implements `Clone` no matter what `B` is. `self.c.clone()` is _not_ fine: we have no information about `C`, so the compiler can't guarantee that it implements `Clone`.

We could fix the problem by adding a type constraint on `C`:

```rust
struct Foo<B, C: Clone> {
  a: u32,
  b: Rc<B>,
  c: C,
}
```

Now `self.c.clone()` will compile. But this constraint means that we can only build a `Foo<B, C>` when `C` implements `Clone`. This is actually a reasonable solution[^what-does-reasonable-mean], but it's not the only reasonable solution. It's also not the one the team implementing `#[derive(Clone)]` chose.

# How `#[derive(Clone)]` is actually implemented

A nifty thing about Rust macros expanding to real Rust code is that we can inspect the expanded Rust code directly. Let's make a new crate and put the following code into `src/lib.rs`:

```rust
use std::rc::Rc;

#[derive(Clone)]
struct Foo<B, C> {
  a: u32,
  b: Rc<B>,
  c: C,
}
```

 Now we can run [`cargo expand`](https://github.com/dtolnay/cargo-expand)[^cargo-expand] to see how `#[derive(Clone)]` actually works:

```rust
#[automatically_derived]
#[allow(unused_qualifications)]
impl<B: ::core::clone::Clone, C: ::core::clone::Clone> ::core::clone::Clone for Foo<B, C> {
  #[inline]
  fn clone(&self) -> Foo<B, C> {
    match *self {
      Foo {
        a: ref __self_0_0,
        b: ref __self_0_1,
        c: ref __self_0_2,
      } => Foo {
        a: ::core::clone::Clone::clone(&(*__self_0_0)),
        b: ::core::clone::Clone::clone(&(*__self_0_1)),
        c: ::core::clone::Clone::clone(&(*__self_0_2)),
      },
    }
  }
}
```

There's a lot of name mangling and weird reference stuff going on, but we can ignore that. The important part is the trait implementation signature:

```rust
impl<B: ::core::clone::Clone, C: ::core::clone::Clone> ::core::clone::Clone for Foo<B, C>
```

This is how `#[derive(Clone)]` guarantees that it can call `self.c.clone()`: it adds a type constraint on `C` so that `Foo<B, C>` only implements `Clone` when `C` implements `Clone`. This is another reasonable solution to the problem: whereas the `struct Foo<B, C: Clone>` implementation above says that you can only create a `Foo<B, C>` when `C` implements `Clone`, this implementation says that you can _create_ a `Foo<B, C>` with any `B` and `C` you like, but that you can only _clone_ your `Foo<B, C>` if `C` implements `Clone`.

But there's something else going on here. This implementation also adds the `B: Clone` constraint, even though we don't need that constraint for `Foo<B, C>` to implement `Clone`. Remember, the `b` field of `Foo<B, C>` is a `Rc<B>`. `b.clone()` will typecheck for any `B`.

This at least explains the error I was getting in my `CloneByRc<NNC>` example in the riddle I started with: Rust complained that `the trait bound NNC: Clone is not satisfied` because the implementation of `#[derive(Clone)]` was adding an `NNC: Clone` type constraint. But why does `#[derive(Clone)]` add this unnecessary constraint?

# The limits of Rust macros

What we'd really want is an implementation that adds type constraints only when we need them. For this example, we'd like something like

```rust
impl<B, C: Clone> Clone for Foo<B, C>
```

The problem is that this requires the macro to inspect the fields of `Foo<B, C>`, figure out which ones are already `Clone`, and only add the type constraint for the other fields. Rust's procedural macros don't have access to the Rust compiler, so the macro would have to implement all of the necessary type checking by itself.

What's worse, this kind of inspection is actually impossible in practice. The input to the macro is just the list of tokens that define `Foo<B, C>`. It only gets syntactic information ("the name of the type of this field is `Rc<B>`") rather than semantic information (the definition of the `Rc` type), so the macro would need to do its type checking given only the names of the types being referenced. The definition of `Foo<B, C>` might include references to arbitrary other types, including types defined elsewhere in the crate, so there's no way the macro can know how to check if the type implements `Clone` given only its name.

# Do we have to inspect the fields?

So we can't check if our fields are guaranteed to implement `Clone`. Can we avoid unnecessary type constraints by being a little more clever? For example, we could imagine an implementation that adds exactly the type constraints it needs:

```rust
impl<B, C> Clone for Foo<B, C> where Rc<B>: Clone, C: Clone
```

Here we've just taken the exact types of the fields that use our generic type parameters and constrained those types to implement `Clone`. This seems like exactly what we want: `Rc<B>: Clone` is true for all `B`, so `C: Clone` is the only nontrivial constraint.

This works for our current example, but it has some unfortunate downsides. First of all, we've just revealed that `Foo<B, C>` uses an `Rc<B>`. Since `b` was a private field of `Foo`, it's a little weird to leak this information.

The bigger downside, though, is that we might start leaking private _types_. Let's take a look at this example:

<table class="two-column-code">
  <tr>
    <td>
      <div>Struct</div>
    </td>
    <td>
      <div>Implementation</div>
    </td>
  </tr>
  <tr>
    <td>
<div class="language-rust highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">struct</span> <span class="n">PrivateFoo</span><span class="o">&lt;</span><span class="n">B</span><span class="o">&gt;</span> <span class="p">{</span>
  <span class="n">b</span><span class="p">:</span> <span class="n">B</span><span class="p">,</span>
<span class="p">}</span>

<span class="k">pub</span> <span class="k">struct</span> <span class="n">Foo</span><span class="o">&lt;</span><span class="n">B</span><span class="p">,</span> <span class="n">C</span><span class="o">&gt;</span> <span class="p">{</span>
  <span class="n">a</span><span class="p">:</span> <span class="nb">u32</span><span class="p">,</span>
  <span class="n">b</span><span class="p">:</span> <span class="n">PrivateFoo</span><span class="o">&lt;</span><span class="n">B</span><span class="o">&gt;</span><span class="p">,</span>
  <span class="n">c</span><span class="p">:</span> <span class="n">C</span><span class="p">,</span>
<span class="p">}</span>




</code></pre></div></div>
    </td>
    <td>
<div class="language-rust highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">impl</span><span class="o">&lt;</span><span class="n">B</span><span class="p">,</span> <span class="n">C</span><span class="o">&gt;</span> <span class="n">Clone</span> <span class="k">for</span> <span class="n">Foo</span><span class="o">&lt;</span><span class="n">B</span><span class="p">,</span> <span class="n">C</span><span class="o">&gt;</span>
<span class="k">where</span>
  <span class="n">PrivateFoo</span><span class="o">&lt;</span><span class="n">B</span><span class="o">&gt;</span><span class="p">:</span> <span class="n">Clone</span><span class="p">,</span>
  <span class="n">C</span><span class="p">:</span> <span class="n">Clone</span><span class="p">,</span>
<span class="p">{</span>
  <span class="k">fn</span> <span class="nf">clone</span><span class="p">(</span><span class="o">&amp;</span><span class="k">self</span><span class="p">)</span> <span class="k">-&gt;</span> <span class="n">Self</span> <span class="p">{</span>
    <span class="n">Foo</span> <span class="p">{</span>
      <span class="n">a</span><span class="p">:</span> <span class="k">self</span><span class="py">.a</span><span class="nf">.clone</span><span class="p">(),</span>
      <span class="n">b</span><span class="p">:</span> <span class="k">self</span><span class="py">.b</span><span class="nf">.clone</span><span class="p">(),</span>
      <span class="n">c</span><span class="p">:</span> <span class="k">self</span><span class="py">.c</span><span class="nf">.clone</span><span class="p">(),</span>
    <span class="p">}</span>
  <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>
    </td>
  </tr>
</table>

Here our private type `PrivateFoo<B>` is now part of the public signature of our `Clone` implementation. This will break compilation: the Rust compiler will throw `error[E0446]: private type PrivateFoo<B> in public interface`. I think this error is a lot worse than the `trait bound NNC: Clone is not satisfied` error we get with the current implementation of `#[derive(Clone)]`, since this error sounds scarier and doesn't seem obviously related to implementing `Clone`. If I saw this error, I'd probably think something was wrong elsewhere in my program and waste a bunch of time trying to track it down[^relative-surprise][^other-scary-consequences].

# The workaround

Ok, so we've looked at a bunch of alternative ways to derive `Clone` and none of them seem like they would actually work. My struct is still broken. How do I fix it?

The answer, it turns out, is pretty straightforward: just implement `Clone` manually instead of deriving it. Unlike `#[derive(Clone)]`, you only need to solve the problem for one struct and you have access to semantic information about all of the types you're using. You can pick the exact type constraints you want.

In fact, I suspect that part of the reason `#[derive(Clone)]` hasn't fixed this issue is that the workaround is so easy.

# Conclusion: why did they do it this way?

By now we've seen a few possible implementations for `#[derive(Clone)]`. The perfect ones are impossible and the possible ones are imperfect. All of them mean that you have to implement `Clone` manually for some types of structs, so choosing between them is a design decision. You might choose to optimize for the most common case or for the case where implementing manually is the most painful, but whichever one you pick you're going to leave somebody wondering why they have to implement `Clone` manually.

Here are the two most plausible implementations:

|Implementation|No type constraints|Type constraints on all type parameters|
|Example|`impl<B, C> Clone for Foo<B, C> { ... }`|`impl<B: Clone, C: Clone> Clone for Foo<B, C> { ... }`|
|Works when|All fields implement `Clone` regardless of the input types|All input types implement `Clone`|
|Breaks when|At least one field might not implement `Clone` for some input type|At least one field implements `Clone` regardless of the input types (for example, an `Rc<T>`), but at least one of that field's input types doesn't implement `Clone`|

Given the situations in which these two implementations break down and the fact that the workaround is so straightforward, I'm not surprised that the authors of `#[derive(Clone)]` chose the "type constraints on all type parameters" approach.

[^clone-is-an-example]: I'm using `Clone` as my example here, but this discussion applies to any derivable trait. If you're not familiar with `Clone`, you can basically think of it as "any object that implements Clone can be deep-copied". Here's [a good explainer](https://hashrust.com/blog/moves-copies-and-clones-in-rust/).

[^procedural-macros]: To be precise, `#[derive(Clone)]` is a [procedural macro](https://doc.rust-lang.org/reference/procedural-macros.html), which means that it takes a [stream of tokens](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html) (the individual words of Rust code that make up the struct definition) and returns a stream of tokens (the words that make up the trait implementation).

[^mixed-levels]: There's a bit of sleight of hand in this psuedocode: I've mixed code from different levels of abstraction. The structure of the trait implementation (`impl Clone`, `fn clone(&self)`) is all literal Rust code that will be emitted as written and checked by the compiler. But `struct_name` is not literal Rust code: it's a variable that holds the name of the struct. In the actual emitted Rust code, `struct_name` will be replaced with, say, `CloneByRc`. Similarly, the `for` loop will not show up in the literal Rust code; it will be replaced by its output. In a real procedural macro, the distinction between "code that is emitted as written" and "code that is evaluated" is made explicit using a [quasi-quote](https://en.wikipedia.org/wiki/Quasi-quotation) function such as the [`quote!` macro](https://docs.rs/quote/1.0.9/quote/index.html). I've left it mixed like this for simplicity.

[^what-does-reasonable-mean]: Or at least, _I_ think it's reasonable. As we'll see, there isn't really a perfect solution here so any solution is going to end up missing some edge cases. Which edge cases you choose to leave out depends on your mental model of which edge cases are most common/most painful to work around. I doubt that anybody has done any meaningful data analysis to measure that in an objective sense, so it doesn't seem like this solution is obviously worse than the one chosen.

[^cargo-expand]: If you're having trouble getting `cargo expand` to run, I have [a blog post](/rustfmt-nightly) just for you.

[^relative-surprise]: While the error message would tell me that this error came from the macro implementation of `Clone`, before going down this rabbit hole I would have had no idea why an implementation of `Clone` would leak a private type.

[^other-scary-consequences]: This is by no means an exhaustive list of what could go wrong with this type of implementation. There's a pretty good discussion of other scary and exciting consequences on this [Github issue](https://github.com/rust-lang/rust/issues/26925), which was the source for most of this blog post.
