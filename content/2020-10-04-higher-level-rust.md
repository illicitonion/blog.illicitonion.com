+++
title = "Higher-level Rust"
+++

# Background

As I've noted in the past, I really enjoy using the Rust programming language. There is a trope that when learning Rust, that you are always fighting the borrow checker, but that at the end of the day the borrow checker is always right. Certainly when I started learning the language, I felt I was frequently fighting against it, but that has largely abated over time.

Rust, however, identifies as a language designed for systems programming, and has many users who deeply care about specifics of things like memory layout, and ensuring anything with cost (e.g. allocating to the heap, or increasing a reference count) be explicit. I rarely work at this level (generally writing higher-level applications where these costs don't much matter most of the time). Most of the times I feel like I'm fighting against the language now come from me not caring about these things. Unfortunately, the language does.

[boats](https://without.boats/) has written [a couple](https://without.boats/blog/notes-on-a-smaller-rust/) of excellent [blog posts](https://without.boats/blog/revisiting-a-smaller-rust/) on the idea of a smaller Rust - a language which shares many of the same core features of Rust (strong enums/pattern matching, RAII, concurrency-safety), but which makes fewer guarantees to improve ergonomics. I've found these very interesting and motivating, though I feel, particularly in the second blog post, they are proposing a much greater departure from Rust than I personally desire.

## How Rust compiles today

[This blog post](https://blog.rust-lang.org/2016/04/19/MIR.html) describes the stages `rustc` currently goes through when compiling; loosely:
1. Lexing, Parsing, Macro expansion, Name resolution, and Feature resolution - turning your source file into an AST.
2. Lowering to HIR - converting the AST into HIR - desugaring things like `for` loops into `loop`s, filling in `impl Trait`s with concrete types, etc.
3. Lowering to MIR - type checking, converting the HIR to MIR - making things look a lot more like control-flow graphs.
4. Lowering the MIR to LLVM IR - borrow checking, and converting to a format LLVM can actually use for code generation.
5. Code generation - actually turning IR into machine code.

I've been toying with the idea of trying to make such a language[^1], and in thinking through some of the practicalities, one thing I haven't yet worked out is whether this language would look more like:
1. A minor variant of Rust (e.g. some feature flags that probably wouldn't be accepted upstream, but which could fit easily into the existing rustc) - minor changes to individual pieces of step 1 in the above pipeline.
2. A close enough cousin of Rust to share the same HIR or MIR - rewriting much of step 1 in the above pipeline, but looking to produce compatible HIR or MIR.
3. Far enough away from Rust to not really benefit from re-using any of the existing rustc.

boats's blog posts suggest they are minded towards 3 - they don't want to use LLVM, and they want to offload many responsibilities to WASM. While they have reasons for this, I'm not sure my desires are so extreme. To help feel out where I think the split should happen for the language I have in mind, I'm going to use this post, which I will probably edit over time, as a list of features I may want in such a language. I'm going to explicitly take Rust as my base-line - these features will be diffs against Rust as it is today, rather than dreaming from scratch. I imagine any time I get annoyed writing some Rust, I will come update this list.

# Feature list

So, things I want which Rust doesn't offer me today, in no particular order:

## 1. Automatic hoisting of temporaries

<details>

I have [previously written](/rust-papercuts-1-please-own-my-temporary/) about some annoyances here, but this requirement is more general. Within a function, I would like temporaries to be auto-promoted to the stack wherever needed (as described in that post).

Additionally, between functions, I would like temporaries to be auto-promoted (either to the calling stack frame, as an implicit hidden bit of "backing" memory in the return value, or to the heap) wherever needed.

Here's an example to illustrate the "between functions" case:
```rust
fn lines() -> std::str::Lines {
    let s = String::new("blah");
    s.lines()
}
```

This doesn't compile because `s` gets dropped at the end of the call to `lines`, but `s` could be moved onto the heap, or into the caller's stack frame to make it compile fine.

### Why won't this make it into Rust?
The first half of this, stack promotion, I think could land in Rust, but the promotion of temporaries across function boundaries is very unlikely to - both "Implicitly allocate" and "Hide data" are very against the Rust ways.

### Could this be a proc macro?
The stack promotion half: Yes, but it would require annotating every place where this happened, potentially multiple times, which feels like a lot of friction.

The inter-function half: I can't imagine a nice way - you'd need to change the return type of the function, and that would not be fun.

### Where could this change happen in rustc?
Probably when lowering to MIR, but possibly before.

</details>

## 2. Configurable autoclone

<details>

Right now, as far as Rust is concerned, there are two types of values: Those which can be implicitly copied (marked with the [`Copy`](https://doc.rust-lang.org/std/marker/trait.Copy.html) marker trait), and those which must be explicitly copied (which implement the [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html) trait, but not the `Copy` marker trait).

This reflects one of Rust's deeply held views: Things which may be expensive should be explicit (and `Clone` can be arbitrarily expensive).

If one frequently uses types such as [`Rc`](https://doc.rust-lang.org/std/rc/struct.Rc.html) and [`Arc`](https://doc.rust-lang.org/std/sync/struct.Arc.html), one finds their code littered with calls to `.clone()`, particularly around closures (where `Copy` types are implicitly copied into the closure, but `Clone` types are not).

I think a language with different priorities could work differently in a number of ways, such as:
1. Adding a marker trait for `AutoClone` types - types which should be considered `Copy`, even though they're not (primarily with smart pointers in mind, such as `Arc`).
2. An opt-in `AutoClone` attribute, allowing crates, modules, or functions to specify which types should implicitly get a `.clone()` added where needed, just as `Copy` types get an implicit `memcpy` added, [as I described in an IRLO post](https://internals.rust-lang.org/t/idea-improving-the-ergonomics-when-using-rc/9293/75).

### Why won't this make it into Rust?
The marker trait idea goes against Rust's ideas of explicitness at call sites - the caller, rather than callee, should be the place making decisions about expensive operations.

The attribute idea potentially could make it into Rust - it's much more like call-site sugar - but feels a bit like splitting the ecosystem into dialects, which is frequently argued against. That said, this is probably worth drafting an RFC for before giving up on.

### Could this be a proc macro?

Ish. [There](https://crates.io/crates/capture) [are](https://crates.io/crates/closet) [some](https://crates.io/crates/clone_all) which are specifically targeted at the closure case, but they still add noise where I don't want any noise.

### Where could this change happen in rustc?

I'm not sure, but definitely not before lowering to HIR.

</details>

## 3. Multi-implementation `impl Trait` return values

<details>

When `impl Trait` was introduced in return-type position, it allowed for both convenience and new capabilities:

Convenience: No longer needing to write out potentially unwieldy return types, e.g.:
```rust
fn make_iter() -> std::iter::Rev<std::str::Chars<'static>> {
    "abc123".chars().rev()
}
```
becomes
```rust
fn make_iter() -> impl Iterator<Item=char> {
    "abc123".chars().rev()
}
```

Capabilities: Being able to return types which were previously un-namable:
```rust
fn make_fn() -> impl Fn() -> bool {
    || true
}
```

However, all values returned must have the exact same type; you can't do something like:
```rust
fn maybe_reverse(rev: bool) -> impl Iterator<Item=char> {
    let it = "abc123".chars();
    if rev {
        return it.rev();
    }
    it
}
```

This is because `impl Trait` really means "This function returns exactly one type which implements this trait, but I'm not going to tell you exactly what it is", rather than "This function returns some value which implements this trait". The really core issue is that the compiler needs to have a single size it can allocate on the stack for the return value.

There are a number of ways people work around this, commonly:
1. Explicitly Boxing a return value (allocating it on the heap, and adding runtime overhead when using it - both things which Rust doesn't like doing implicitly):
   ```rust
     fn maybe_reverse(rev: bool) -> Box<dyn Iterator<Item=char>> {
       let it = "abc123".chars();
       if rev {
           return Box::new(it.rev());
       }
       Box::new(it)
   }
   ```
2. Returning an `enum` with a variant-per-return-type which delegates to the undelying value:
   ```rust
   enum Anon<Type1, Type2> {
       Value1(Type1),
       Value2(Type2),
   }

   impl <Type1: std::iter::Iterator<Item=char>, Type2: std::iter::Iterator<Item=char>> Iterator for Anon<Type1, Type2> {
       type Item = char;

       fn next(&mut self) -> Option<Self::Item> {
           match self {
               Self::Value1(v) => v.next(),
               Self::Value2(v) => v.next(),
           }
       }
   }

   fn maybe_reverse(rev: bool) -> impl Iterator<Item=char> {
       let it = "abc123".chars();
       if rev {
           return Anon::Value1(it.rev());
       }
       Anon::Value2(it)
   }
   ```

   For which there exist proc macros such as [`auto_enums`](https://crates.io/crates/auto_enums) to make this more concise for a closed set of traits, but they have limitations:
   ```rust
   #[auto_enum(Iterator)]
   fn maybe_reverse(rev: bool) -> impl Iterator<Item=char> {
       let it = "abc123".chars();
       if rev {
           it.rev()
       } else {
           it
       }
   }
   ```

There has been fairly extensive discussion around "auto-enums" ([informal RFC](https://github.com/rust-lang/rfcs/issues/2414), some of many [IRLO](https://internals.rust-lang.org/t/extending-impl-trait-to-allow-multiple-return-types/7921) [posts](https://internals.rust-lang.org/t/pre-rfc-returning-automatically-generating-impl-trait/13090)).

### Why won't this make it into Rust?
It may! I hope it does! I'm personally hoping for a sufficiently concise syntax like:
```rust
fn maybe_reverse(rev: bool) -> impl enum Iterator<Item=char> {
    let it = "abc123".chars();
    if rev {
        it.rev()
    } else {
        it
    }
}
```

Something less concise than that (e.g. having to annotate each return value), I could live with, but I really don't want to have to think about that much... Alternatively, if this doesn't end up happening, perhaps an auto-boxing proc macro would be sufficient.

### Could this be a proc macro?
To automatically `Box`, probably. To automatically generate an `enum` which implements arbitrary traits? No.

### Where could this change happen in rustc?
Very early on, this is very sugar-y - probably before HIR, but definitely before MIR.

</details>

## Grab-bag still to be written up

* Object safety - Rust's object safety rules are occasionally frustrating, but I haven't done enough research to talk about them yet.
* Stuff which is coming in Rust, just not yet: [GAT](https://github.com/rust-lang/rust/issues/44265), [Specialisation](https://github.com/rust-lang/rust/issues/31844).

<!---

## 1. The thing

<details>

Blurb

### Why won't this make it into Rust?

### Could this be a proc macro?

### Where could this change happen in rustc?

</details>

-->


[^1]: I've never written a programming language before, so this would be both fun as a learning experience, and probably produce some awful results that no one would actually want to use.
