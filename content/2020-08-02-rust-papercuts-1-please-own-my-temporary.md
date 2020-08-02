+++
title = "Rust Papercuts #1: Please own my temporary"
+++

I really enjoy writing Rust - it's a language which provides a lovely balance between elegance and pragmatism.

However, every now and then I try to write some code, and need to add a layer of complexity which it would be lovely if the language sugared[^1] away for me.

This is the first post in a series of small frustrations I'd love not to need to work around myself. Most of them are things which I think could reasonably be fixed in the language, and I'll try to explore possible solutions - maybe we'll even end up with some RFCs!

Today, I'll be talking about ownership of temporaries in blocks.

Consider the following code:
```rust
fn main() {
    let mut snack = "Cheese";
    if true {
        snack = &format!("{}burger", snack);
    }
    assert_eq!(snack, "Cheeseburger");
}
```

This is something I frequently want to do - have a reference, conditionally make some modification which requires me to make a brand new owned value, but then only use it as a reference.

The rust compiler, correctly, complains about this:

```
error[E0716]: temporary value dropped while borrowed
  |
  |         snack = &format!("{}burger", snack);
  |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^- temporary value is freed at the end of this statement
  |                  |
  |                  creates a temporary which is freed while still in use
  |     }
  |     assert_eq!(snack, "Cheeseburger");
  |     ---------------------------------- borrow later used here
  |
  = note: consider using a `let` binding to create a longer lived value

```

Nothing owns the `String` which is being produced by `format!`, so it gets dropped straight after being created, and we can't hold on to a reference to it. Reasonable, but annoying.

## Possible solutions

There are a number of ways I can fix this in my own code, but they all leave something to be desired:

### Own the original

Often, I could instead have owned the original value, rather than used a reference to it. This may come at a cost (cloning the value when we don't really _need_ to), but would often work:
```rust
fn main() {
    let mut snack = "Cheese".to_owned();
    if true {
        snack = format!("{}burger", snack);
    }
    assert_eq!(&snack, "Cheeseburger");
}
```

### Use `Cow` (or a similar wrapper)

[`std::borrow::Cow`](https://doc.rust-lang.org/std/borrow/enum.Cow.html) is an `enum` which allows you to treat a single variable as _maybe_ borrowed or _maybe_ owned - it also allows you to make an owned version from a borrowed one if needed (which we don't here):
```rust
use std::borrow::Cow;

fn main() {
    let mut snack = Cow::Borrowed("Cheese");
    if true {
        snack = Cow::Owned(format!("{}burger", snack));
    }
    assert_eq!(snack, "Cheeseburger");
}
```

This is nice and explicit - the reader can see that they don't necessarily know whether the data is owned, and if they _do_ want an owned value, they can _conditionally_ do the cloning, rather than unconditionally doing so up-front.

However, this approach has two drawbacks:
1. The cognitive overhead of having to write this out, and read it - particularly in the case where you're only treating the data as a reference.
2. There is a (probably very small) runtime overhead, as extra memory needs to be allocated for the `enum` wrapper, and conditionals need to be evaluated when accessing the data.

Rust prides itself on being a language of "zero-cost abstractions" - all of the nice properties it gives you _don't_ come at a cost at runtime, and using `Cow` like this introduces a cost, so while this is a perfectly viable approach to manually keep around a temporary, it is unlikely to become something automatically done for you.

### Conditionally use a backing variable

We can create a variable in the same scope as `snack` which we only use if we hit the branch, to own the underlying data. Effectively, we can _hoist_ ownership up to the same scope as `snack`:
```rust
fn main() {
    let mut snack = "Cheese";
    let _backing;
    if true {
        _backing = format!("{}burger", snack);
        snack = &_backing;
    }
    assert_eq!(snack, "Cheeseburger")
}
```

This has a lower cognitive overhead to me than the `Cow` solution - the considerations are effectively the same, but they're localised - if I see a use of `snack` I always know it's a reference, and I use it as one. I only need to think about this `_backing` field when I'm setting a new value, which is kind of annoying, but ok.

We've gotten rid of the (small) runtime overhead of adding an `enum` into the mix, at the expense of no longer being able to do conditional cloning if we need an owned version. That feels like a fair trade-off - if you want conditional cloning behaviour, you should probably be explicit about using a `Cow`.

I feel that this is an approach which we could potentially bake into Rust itself. Similarly to how [non-lexical lifetimes](https://doc.rust-lang.org/edition-guide/rust-2018/ownership-and-lifetimes/non-lexical-lifetimes.html) (often referred to as "NLL") made the borrow checker less strict, we could apply a similar approach here. The compiler could  notice that we're creating a temporary whose lifetime is directly tied to another variable, and generate the extra code for us. It's worth noting that this is different from NLL, in that NLL didn't change any _semantics_ of code - values were still created and dropped at the same places, whereas with this hoisting, we _are_ changing where values are dropped.

#### Evolving into a language feature

To test this out, I put together a procedural macro, [hoist_temporaries](https://github.com/illicitonion/hoist_temporaries), which automates this hoisting, so that you can write the code from the very first example, with just a couple of added attributes:
```rust
#[hoist_temporaries::hoist_temporaries]
fn main() {
    #[hoist_temporaries::hoist]
    let mut snack = "Cheese";
    if true {
        snack = &format!("{}burger", snack);
    }
    assert_eq!(snack, "Cheeseburger")
}
```

It has a few drawbacks, notably:
1. Macros don't know enough to know whether the value you're assigning is a temporary or not, so the backing variable will take ownership of whatever you're assigning. This means that you can't re-use it after the assignment, like `cheeseburger` in this example:
   ```rust
   #[hoist_temporaries::hoist_temporaries]
   fn main() {
       let cheeseburger = String::from("Cheeseburger");

       #[hoist_temporaries::hoist]
       let mut snack = "Cheese";
       if true {
           snack = &cheeseburger;
       }
       assert_eq!(snack, &cheeseburger)
   }
   ```
2. You need to explicitly put an attribute on both the function, and the specific temporaries you're hoisting.

   Ideally, functions which happen to contain hoisted variables wouldn't need an attribute.

   Having to mark the variables in some way as being hoisted may be long-term desirable (to be explicit that something funny is going on), though I'd rather get rid of this need entirely.

   An interesting alternative could be to put the attribute on the value being hoisted, rather than the variable it's for; that could also help solve the temporary-detection problem. Something like:
   ```rust
   fn main() {
       let mut snack = "Cheese";
       if true {
           #[hoist_temporaries::hoist]
           snack = &format!("{}burger", snack);
       }
       assert_eq!(snack, "Cheeseburger")
   }
   ```
   or even:
   ```rust
   fn main() {
       let mut snack = "Cheese";
       if true {
           snack = &hoist format!("{}burger", snack);
       }
       assert_eq!(snack, "Cheeseburger")
   }
   ```


   To get rid of either of these required attributes, though, we'd definitely need to be better integrated with (i.e. part of) the compiler.

3. I invented a syntax which, though inspired by the feel of the language, isn't Rust. Arbitrary statements can't be given attributes in normal code, and it would be reasonable for the parser to start rejecting this code in the future. This approach seemed to be the cleanest way to note a property of a variable, while still being parsable as Rust (I would perhaps have preferred `let hoisted mut snack = "Cheese";`, but that made the parser unahppy).

   I also implemented a version where you note the variables being hoisted on the function attribute, like:
   ```rust
   #[hoist_temporaries::hoist_temporaries(snack)]
   fn main() {
       let mut snack = "Cheese";
       if true {
           snack = &format!("{}burger", snack);
       }
       assert_eq!(snack, "Cheeseburger")
   }
   ```

   This was slightly more in keeping with Rust proper, but I didn't like the distance between the variable and the information about it.

## Next steps

I'd love to hear what others thing. Is this a problem you've run into? Would any of the things I've described above help you? Do they seem somehow awful, or have flaws I haven't considered? Have you solved this problem another way?

I wouldn't necessarily suggest that the [hoist_temporaries](https://crates.io/crates/hoist_temporaries) crate is actually useful (though feel free to use it, and feedback is very welcome!), but it was an interesting way to explore the practical issues in the space.

I'd be interested in maybe putting together an RFC for solving this problem in Rust proper at some point. It is related to the work described in [RFC 66](https://rust-lang.github.io/rfcs/0066-better-temporary-lifetimes.html), but unfortunately that was written in quite a vague open-ended way in the early days of RFCs, and also appears to be somewhat stalled. If anyone is interested in working with me on this, please do reach out!

[^1]: "Sugar" is when the compiler takes the code you wrote, and expands it to some more complex code for you, so that you get all of the benefits of the expanded code, without having to make your code more verbose.
