---
title: "Understanding #[derive(Clone)] in Rust"
date: 2021-08-11T08:00:00-07:00
classes: wide
editors:
  - Bianca Homberg
categories:
  - programming
tags:
  - rust
---

This post assumes that you have an entry-level familiarity with Rust: you've fought with the borrow checker enough to start to internalize some of its model; you've defined structs, implemented traits on those structs, and derived implementations of common traits using macros; you've seen [trait bounds](https://doc.rust-lang.org/rust-by-example/generics/bounds.html) and maybe used one or two. Most importantly, you know what the `Clone` trait is and why you would want to use it[^clone-is-an-example].

# A Rust riddle

Now that we've set expectations, let's start with a Rust riddle. This code compiles:

```rust
use std::rc::Rc;

trait NotNecessarilyClone {}

#[derive(Clone)]
struct CloneByRc<NNC> where NNC: NotNecessarilyClone {
  nnc: Rc<NNC>,
}
```

This on its own is not particularly surprising. We have a trait `NotNecessarilyClone`. Things that implement this trait might or might not also implement `Clone`. We have a struct `CloneByRc` that stores an object that implements `NotNecessarilyClone`, but it stores it behind an [`Rc`](https://doc.rust-lang.org/std/rc/struct.Rc.html)[^what-is-rc] so that `CloneByRc` can implement `Clone`. This is kind of the point of an `Rc`: `Rc<T>` [implements `Clone`](https://doc.rust-lang.org/std/rc/struct.Rc.html#impl-Clone) for all `T`. It takes something that isn't normally safe to clone and hides it behind a reference counter so that cloning just means incrementing the reference counter.

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

At first, I was confused about why this failed to compile. Why does Rust complain that `the trait Clone is not implemented for NNC`? I only want `CloneByRc<NNC>` to implement `Clone`. The whole point of using an `Rc` was so that `NNC` wouldn't *need* to implement `Clone`.

Once I understood the compilation error, I was confused about why the original code compiled at all. If `CloneByRc<NCC>` doesn't actually implement `Clone`, why didn't the original code throw an error when I tried to `#[derive(Clone)]`?

Eventually, I figured out both the compilation error and the successful compilation of the original code, which is why I'm writing this blog post.

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

This is how `#[derive(Clone)]` guarantees that it can call `self.c.clone()`: it adds a type constraint on `C` so that `Foo<B, C>` only implements `Clone` when `C` implements `Clone`[^why-fully-qualified]. This is another reasonable solution to the problem: whereas the `struct Foo<B, C: Clone>` implementation above says that you can only create a `Foo<B, C>` when `C` implements `Clone`, this implementation says that you can _create_ a `Foo<B, C>` with any `B` and `C` you like, but that you can only _clone_ your `Foo<B, C>` if `C` implements `Clone`.

But there's something else going on here. This implementation also adds the `B: Clone` constraint, even though we don't need that constraint for `Foo<B, C>` to implement `Clone`. Remember, the `b` field of `Foo<B, C>` is a `Rc<B>`. `b.clone()` will typecheck for any `B`.

This at least explains the compilation error I was getting in my `CloneByRc<NNC>` example in the riddle I started with: Rust complained that `the trait bound NNC: Clone is not satisfied` because the implementation of `#[derive(Clone)]` was adding an `NNC: Clone` type constraint. But why does `#[derive(Clone)]` add this unnecessary constraint?

# The limitations of Rust macros

What we'd really want is an implementation that adds type constraints only when we need them. For this example, we'd like something like

```rust
impl<B, C: Clone> Clone for Foo<B, C>
```

Why doesn't `#[derive(Clone)]` produce this implementation instead?

The problem is that this implementation requires the macro to inspect the fields of `Foo<B, C>`, figure out which ones are already `Clone`, and only add the type constraint for the other fields. The short answer is that Rust's procedural macros just aren't powerful enough to do this. They don't have access to the Rust compiler, so our optimal `#[derive(Clone)]` macro would have to implement all of the necessary type checking by itself.

The long answer is that this kind of inspection is actually impossible in practice. The input to the macro is just the list of tokens that define `Foo<B, C>`. It only gets syntactic information ("the name of the type of this field is `Rc<B>`") rather than semantic information (the definition of the `Rc` type), so the macro would need to do its type checking given only the names of the types being referenced. The definition of `Foo<B, C>` might include references to arbitrary other types, including types defined elsewhere in the crate, so there's no way the macro can know how to check if the type implements `Clone` given only its name.

# Do we have to inspect the fields?

So we can't check if our fields are guaranteed to implement `Clone`. But there's still got to be a way to avoid unnecessary type constraints, right? What if we try being a little more clever? For example, we could imagine an implementation that adds exactly the type constraints it needs:

```rust
impl<B, C> Clone for Foo<B, C> where Rc<B>: Clone, C: Clone
```

Here we've just taken the exact types of the fields that use our generic type parameters and constrained those types to implement `Clone`. This seems like exactly what we want: `Rc<B>: Clone` is true for all `B`, so `C: Clone` is the only nontrivial constraint.

This works for our current example, but it has some unfortunate downsides. First of all, our implementation now reveals that `Foo<B, C>` uses an `Rc<B>`. Since `b` was a private field of `Foo`, it's a little weird to leak this information.

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

Now our private type `PrivateFoo<B>` is part of the public signature of our `Clone` implementation. This will break compilation: the Rust compiler will throw `error[E0446]: private type PrivateFoo<B> in public interface`[^other-scary-consequences]. I think this error is a lot worse than the `trait bound NNC: Clone is not satisfied` error we get with the current implementation of `#[derive(Clone)]`, since this error sounds scarier and doesn't seem obviously related to implementing `Clone`. If I saw this error, I'd probably think something was wrong elsewhere in my program and waste a bunch of time trying to track it down[^relative-surprise].

# The workaround

Ok, so we've looked at a bunch of alternative ways to derive `Clone` and none of them seem like they would actually work. My struct is still broken. How do I fix it?

The answer, it turns out, is pretty straightforward: just implement `Clone` manually instead of deriving it. Unlike `#[derive(Clone)]`, you only need to solve the problem for one struct and you have access to semantic information about all of the types you're using. You can pick the exact type constraints you want.

In fact, I suspect that part of the reason `#[derive(Clone)]` hasn't fixed this issue is that the workaround is so easy.

# Conclusion

The answer to the first part of my riddle ("why do I get an error message complaining that `the trait Clone is not implemented for NNC`?") is that the implementation of `Clone` produced by `#[derive(Clone)]` adds a type constraint `NNC: Clone`. The answer to the second part ("why did the declaration of `CloneByRc<NNC>` compile if `CloneByRc<NNC>` isn't actually guaranteed to be `Clone`?") also comes down to that type constraint: because of the type constraint, `#[derive(Clone)]` doesn't actually guarantee that `CloneByRc<NNC>` will be `Clone` for all `NNC`. It only guarantees that `CloneByRc<NNC>` will be `Clone` whenever `NNC` is `Clone`.

Seeing that type constraint was the first step on our tour through many hypothetical implementations for `#[derive(Clone)]`. Because of the limitations on Rust's derive macros, each of these implementations has a downside: the perfect ones are impossible and the possible ones are imperfect. For any implementation that could be written in Rust today, you'll have to implement `Clone` manually for some types of structs. You might choose to optimize for the most common case or for the case where implementing manually is the most painful, but whichever one you pick you're going to leave somebody wondering why they have to implement `Clone` manually.

Here are the two most plausible implementations:

|Implementation|No type constraints|Type constraints on all type parameters|
|Example|`impl<B, C> Clone for Foo<B, C> { ... }`|`impl<B: Clone, C: Clone> Clone for Foo<B, C> { ... }`|
|Works when|All fields implement `Clone` regardless of the input types|All input types implement `Clone`|
|Breaks when|At least one field might not implement `Clone` for some input type|At least one field implements `Clone` regardless of the input types (for example, an `Rc<T>`), but at least one of that field's input types doesn't implement `Clone`|

Given the situations in which these two implementations break down and the fact that the workaround is so straightforward, I'm not surprised that the authors of `#[derive(Clone)]` chose the "type constraints on all type parameters" approach.

[^clone-is-an-example]: I'm using `Clone` as my example in this blog post, but this discussion applies to any derivable trait. If you're not familiar with `Clone`, here's [a good explainer](https://hashrust.com/blog/moves-copies-and-clones-in-rust/).

[^what-is-rc]: Short for 'Reference Counted'. An `Rc` stores a pointer to an object that lives on the heap and a count of the number of live references to that object. When you create a new reference to the object, the `Rc` increments the counter. When you drop a reference to the object, the `Rc` decrements the counter. When the counter drops to 0 (when the last reference to the object is dropped) the object itself is dropped. This means that you can keep a bunch of references to the same underlying object instead of making multiple copies of the object (which might break the object's functionality if it's something like a shared counter or an HTTP client).

[^procedural-macros]: To be precise, `#[derive(Clone)]` is a [procedural macro](https://doc.rust-lang.org/reference/procedural-macros.html), which means that it takes a [stream of tokens](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html) (the individual words of Rust code that make up the struct definition) and returns a stream of tokens (the words that make up the trait implementation).

[^what-does-reasonable-mean]: Or at least, _I_ think it's reasonable. As we'll see, there isn't really a perfect solution here so any solution is going to end up missing some edge cases. Which edge cases you choose to leave out depends on your mental model of which edge cases are most common/most painful to work around. I doubt that anybody has done any meaningful data analysis to measure that in an objective sense, so it doesn't seem like this solution is obviously worse than the one chosen.

[^cargo-expand]: If you're having trouble getting `cargo expand` to run, I have [a blog post](/rustfmt-nightly) just for you.

[^why-fully-qualified]: If you're wondering why the expanded code uses the fully-qualified trait name `::core::clone::Clone` instead of just `Clone`, it has to do with the way Rust's procedural macros work. When the compiler invokes the macro, the macro returns a block of Rust code that the compiler just inserts directly in-place at the macro's invocation site. The compiler doesn't provide any namespacing for the emitted code, so the macro emits fully-qualified names to avoid name collisions. If the macro just used `Clone` and the file happened to import some other trait named `Clone`, the type checker would expect the emitted code to implement that other `Clone` instead of `::core::clone::Clone`.

[^other-scary-consequences]: This is by no means the only thing that could go wrong with this type of implementation. There's a pretty good discussion of other scary and exciting consequences on this [Github issue](https://github.com/rust-lang/rust/issues/26925), which was the source for most of this blog post.

[^relative-surprise]: While the error message would tell me that this error came from the macro implementation of `Clone`, before going down this rabbit hole I would have had no idea why an implementation of `Clone` would leak a private type.
