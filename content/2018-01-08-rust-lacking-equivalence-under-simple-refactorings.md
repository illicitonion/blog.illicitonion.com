+++
title = "Rust: Lacking equivalence under simple refactorings"
+++

I've been writing a fair bit of Rust recently. I'm new to the language, and it's been alternating between making me very happy, and making me very frustrated. One of the things I've been finding frustrating is the number of places that I expect two pieces of code to be equivalent where they are not. Here's some sample code, which I'm going to use in a few examples:

```rust
#[derive(Debug)]
struct Foo {}

impl Foo {
    fn foo(&self) -> &Foo {
        self
    }

    fn bar(&mut self) -> &Foo {
        self
    }

    fn baz(&mut self) -> &mut Foo {
        self
    }
}

fn call_one() {
    let bar = Foo{}.bar();
    println!("{:?}", bar);
}

fn call_two() {
    let mut foo = Foo{};
    let bar = foo.bar();
    println!("{:?}", bar);
}

fn call_three() {
    let foo = Foo{}.foo();
    println!("{:?}", foo);
}

fn call_four() {
    let mut f = Foo{};
    let baz = f.baz();
    println!("{:?}", baz);
    let bar = f.bar();
    println!("{:?}", bar);
}

fn call_five() {
    let mut f = Foo{};
    println!("{:?}", f.baz());
    println!("{:?}", f.bar());
}

fn call_six() {
    let mut f = Foo{};
    {
        let baz = f.baz();
        println!("{:?}", baz);
    }
    let bar = f.bar();
    println!("{:?}", bar);
}
```

First, let's look at `call_one` and `call_two`. These look like they should be identical code; the simple application of an "inline" refactoring (or "extract variable", if going in the other direction). But in reality, `call_two` is fine, but the borrow checker is unhappy with `call_one`:

```
error[E0597]: borrowed value does not live long enough
  --> src/main.rs:19:26
   |
19 |     let bar = Foo{}.bar();
   |               -----      ^ temporary value dropped here while still borrowed
   |               |
   |               temporary value created here
20 |     println!("{:?}", bar);
21 | }
   | - temporary value needs to live until here
   |
   = note: consider using a `let` binding to increase its lifetime
```

So what's going on here? When you declare a variable with `let`, its lifetime begins, and it remains live until the variable it's bound to goes out of scope. But if you never bind an expression to a variable with `let`, its lifetime immediately ends, and it gets dropped. There seem to be some special-cases where this is transparently handled for you (look at `call_three` - it's exactly the same as `call_one` which the borrow checker finds problematic, but the borrow checker is fine with this one. The only difference is that `bar` borrows self mutably, whereas `foo` doesn't. I assume that internally, the compiler is somehow promoting `foo` to be an owned type, or creating a hidden temporary which owns the `Foo`. It would be nice if the same courtesy could be afforded to more mutable references until something actually tries to violate constraints, like actually modifying the reference, rather than just preventing anonymous mutable borrows entirely).  The big place I've seen this be a problem in real life is with the builder pattern; if I want to populate my builder with some arguments, and then build from it more than once, I need three statements: to assign the un-populated builder, to populate it, and to build it, where I'd really like two: to assign the populated builder, and to build it.

There is work underway in the language to solve part of this problem, with something called [non-lexical lifetimes](https://github.com/rust-lang/rust-roadmap/issues/16) (NLL). Take a look at `call_four`, `call_five`, and `call_six`. Again, these look like they should be equivalent, but `call_four` makes the borrow checker unhappy.

```
error[E0499]: cannot borrow `f` as mutable more than once at a time
  --> src/main.rs:39:15
   |
37 |     let baz = f.baz();
   |               - first mutable borrow occurs here
38 |     println!("{:?}", baz);
39 |     let bar = f.bar();
   |               ^ second mutable borrow occurs here
40 |     println!("{:?}", bar);
41 | }
   | - first borrow ends here
```

This time, the borrow checker is unhappy for the opposite reason: a variable's lifetime is bounded by its scope, and its scope continues until the end of the block in which it was declared, so `baz`, which mutably borrows `f`, precludes `bar` from being able to borrow it, even though `baz` could be analysed by the compiler to be dead. You can work around this by avoiding the assignment, as in `call_five`, or by introducing an inner scope, after which `baz` gets dropped, as in `call_six`. The NLL work will address this, and perform the liveness analysis to render these workarounds unnecessary.

Having to re-structure code which otherwise feels like it should be equivalent to satisfy the borrow checker is pretty frustrating, and occasionally means sacrificing clarity. It's great to see work being done to improve this situation, and I hope it continues to go further!
