+++
title = "Match in TypeScript"
+++

## Sum Types and Pattern Matching

I'm a big fan of compilers telling me I've made a mistake in my code. I'd much rather a compiler tell me early on "this is wrong" than hoping to find out later on (e.g. while running tests, or when a real user is using my project).

One of my favourite constructs for helping with is is union types (what Rust calls `enum`s), and exhaustive pattern matching. These let us say things like:

```rust
// Define the three possible greetings.
enum Greeting {
    Hello,
    Howdy,
    Sup,
}

let greeting = Greeting::Sup;

// Do something different for each option.
match greeting {
    Greeting::Hello => { println!("The greeting is relatively formal."); },
    Greeting::Howdy => { println!("The greeting is folksy."); },
    Greeting::Sup => { println!("The greeting is informal."); },
};
```

In my code I know exactly what values I need to consider. I don't need to worry about someone passing `"Hi"` to my code. I don't need to worry about handling an empty string. I know the three cases I need to consider, and the compiler has my back.

This is not that different from using a `switch` statement or a series of `if-else` statements in another language. The key difference is the exhaustiveness.

If I forgot to handle `Sup`:

```rust
match greeting {
    Greeting::Hello => { println!("The greeting is relatively formal."); },
    Greeting::Howdy => { println!("The greeting is folksy."); },
};
```

The compiler would tell me:

```
error[E0004]: non-exhaustive patterns: `Greeting::Sup` not covered
  --> src/main.rs:11:11
   |
11 |     match greeting {
   |           ^^^^^^^^ pattern `Greeting::Sup` not covered
   |
note: `Greeting` defined here
  --> src/main.rs:3:10
   |
 3 |     enum Greeting {
   |          ^^^^^^^^
...
 6 |         Sup,
   |         --- not covered
   = note: the matched value is of type `Greeting`
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
   |
13 ~         Greeting::Howdy => { println!("The greeting is folksy."); },
14 ~         Greeting::Sup => todo!(),
   |

For more information about this error, try `rustc --explain E0004`.
```

On its own, this is already useful, but the place it really shines is when we _change_ code. Let's imagine I add a new greeting:

```rust
// Define the _four_ possible greetings.
enum Greeting {
    Hello,
    Howdy,
    Sup,
    Heyhey,
}
```

Immediately when I try to compile my code, I'll get exactly that same error telling me: you're not handling one of the greetings - you need to fix it:

```
error[E0004]: non-exhaustive patterns: `Greeting::Heyhey` not covered
  --> src/main.rs:12:11
   |
12 |     match greeting {
   |           ^^^^^^^^ pattern `Greeting::Heyhey` not covered
   |
note: `Greeting` defined here
  --> src/main.rs:3:10
   |
 3 |     enum Greeting {
   |          ^^^^^^^^
...
 7 |         Heyhey,
   |         ------ not covered
   = note: the matched value is of type `Greeting`
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
   |
15 ~         Greeting::Sup => { println!("The greeting is informal."); },
16 ~         Greeting::Heyhey => todo!(),
   |

For more information about this error, try `rustc --explain E0004`.
```

One more detail I like about how Rust handles this is that `match` is an _expression_ not a _statement_. This means it can _evaluate_ to some value, and that value can be used (e.g. assigned to a variable). For instance I can write:

```rust
let description = match greeting {
    Greeting::Hello => "relatively formal",
    Greeting::Howdy => "folksy",
    Greeting::Sup => "informal",
};
println!("The greeting is {}.", description);
```

Were `match` a statement, we wouldn't be able to use it like a variable - each case would need to _do_ something (e.g. print to the terminal, or assign a value to a variable), rather than _evaluating to_ something.

## Enter TypeScript

I've been writing some TypeScript recently, and have been pleasantly surprised that basic[^1] sum types are pretty easy to use in TypeScript:

```ts
type Greeting = "Hello" | "Howdy" | "Sup";
```

With this construct I get similar type-checking benefits to the `enum` above - I don't need to worry about handling a `"Hi"` or an empty string (assuming I'm using types pervasively enough in my codebase). But I don't get `match`.

Out of the box, what I get is `switch`. I can write:

```ts
let description;
switch (greeting) {
    case "Hello":
        description = "relatively formal";
        break;
    case "Howdy":
        descirption = "folksy";
        break;
}
console.log(`The greeting is ${description}.`);
```

There are some things to like about this! If I add a `case "Goodbye"`, I'll get an error. Thank you TypeScript!

But there are also some things to dislike about it:
1. My `description` variable has to be mutable (declared with `let` not `const`) because `switch` is a statement not an expression, and I'm assigning to it in different branches.
2. I missed the `"Sup"` value and TypeScript didn't tell me about it. There's no assumption that my `switch` _should_ be exhaustive, so it's not a error (or even warning) for it not to be.
3. I need to remember all of those `break;` statements. If I forget one, that's a bug.
4. If I forget to set `description` (e.g. above I have a typo in the variable name in the `"Howdy"` branch), TypeScript doesn't tell me `description` sometimes isn't set. There's nothing to ensure I always actually set a value for `description`.

Now, there's [a TC39 proposal around pattern matching](https://github.com/tc39/proposal-pattern-matching) in the works, but it's relatively early in its journey. I was curious in the mean time how close we could get _without much work_. I ended up with this function:

```ts
type Mapping<T extends string | number | symbol, V> = Record<T, V | ((t: T) => V)>;

function match<T extends string | number | symbol, V>(key: T, mapping: Mapping<T, V>): V {
    const v = mapping[key];
    if (typeof v === "function") {
        return v(key);
    } else {
        return v as V;
    }
}
```

I don't hate this. It has a lot of limitations - it's in no means a full `match` implementation. But it _is_ useful.

Let's take a look at some examples.

We can do our simple exhaustive switch-like-thing to produce a value:

```ts
const mapping = {
    "Hello": "relatively formal",
    "Howdy": "folksy",
    "Sup": "informal",
};
const description = match("Hello", mapping);
console.log(`The greeting is ${description}.`);
```

We can run some arbitrary code when certain branches are hit (and without needing to hard-code the value in the function):

```ts
const mapping = {
    "Hello": "relatively formal",
    "Howdy": "folksy",
    "Sup": (v: Greeting) => {
        console.error(`Warning: Saw deprecated warning ${v}`);
        return "informal";
    },
};
const description = match("Hello", mapping);
console.log(`The greeting is ${description}.`);
```

We can do these things inline rather than with a standalone `mapping` variable:

```ts
const greeting: Greeting = "Hello";
const description = match(greeting, {"Hello": "relatively formal", "Howdy": "folksy", "Sup": "informal"});
console.log(`The greeting is ${description}.`);
```

But if we do, something interesting happens! We actually get an error in this case:

```
Object literal may only specify known properties, and '"Howdy"' does not exist in type 'Mapping<"Hello", string>'.
```

This it TypeScript telling us: Hey! You're giving me possible values for this match _but I know they're not used_! This is called [flow-sensitive typing](https://en.wikipedia.org/wiki/Flow-sensitive_typing) - TypeScript can see that even though we declared the `greeting` variable to be of type `Greeting`, it _knows_ it only ever has the value `"Hello"`.

We could change our code in a couple of ways if we wanted to get rid of this warning from our artificially simple example, e.g.

```ts
let greeting: Greeting = "Hello";
// Show TypeScript we could use any of the values:
if (Math.random() > 0.7) {
    greeting = "Howdy";
} else if (Math.random() > 0.3) {
    greeting = "Sup";
}
const description = match(greeting, {"Hello": "relatively formal", "Howdy": "folksy", "Sup": "informal"});
console.log(`The greeting is ${description}.`);
```

or tell TypeScript to consider the full type of `Greeting` rather than using its own flow-typing-narrowed type:

```ts
const greeting: Greeting = "Hello";
const description = match<Greeting, string>(greeting, {"Hello": "relatively formal", "Howdy": "folksy", "Sup": "informal"});
console.log(`The greeting is ${description}.`);
```

But actually, this error could be useful! Remember my first statement: "I'm a big fan of compilers telling me I've made a mistake in my code.". If I wrote code to handle all of `"Hello"`, `"Howdy"`, and `"Sup"`, but my value can only ever be `"Hello"`, maybe I've made a mistake somewhere. Maybe I didn't need to handle `"Sup"` and I can delete the code I wrote to handle it. Maybe it's _surprising_ to me that the value can never be `"Sup"` and that could highlight an error somewhere else in my code I need to fix.

For example, maybe I wrote this code:

```ts
function characteriseGreeting(userInput: string): string {
    console.log(`Characterising ${userInput}`);
    const greeting: Greeting = "Hello"; // TODO: Actually parse userInput
    return match(greeting, {"Hello": "relatively formal", "Howdy": "folksy", "Sup": "informal"});
}

console.log(`The greeting is ${characteriseGreeting("Sup")}`);
```

The compiler just reminded me about my TODO. But it _also_ would have told me this if I hadn't bothered to write the TODO at all:

```ts
function characteriseGreeting(userInput: string): string {
    console.log(`Characterising ${userInput}`);
    const greeting: Greeting = "Hello";
    return match(greeting, {"Hello": "relatively formal", "Howdy": "folksy", "Sup": "informal"});
}

console.log(`The greeting is ${characteriseGreeting("Sup")}`);
```

This is exactly the kind of thing I want a compiler to tell me about! Much better than finding a bug in production where I completely ignore the user input.

## Limitations

I've found this little function useful in a real codebase I was working on. But it has a lot of limitations compared to e.g. Rust's fully-featured built-in `match` keyword. Let's look at a few:

### Range types and guard clauses

In Rust I could write something like:

```rust
let value: u8 = 7;
match value {
    0..=3 => { println!("Small"); },
    4..=15 => { println!("Medium"); },
    16.. => { println!("Large"); },
}
```

That is, I can cover _ranges_ of values with a single branch. I can imagine how we could build a similar list of predicates to check in TypeScript, but we'd probably lose exhaustiveness checking as a result (unless we require every `match` to have some default value, which also undermines exhaustiveness checking).

In Rust I could also write additional guard clauses on a branch:

```rust
let value: u8 = 7;
match value {
    x if x % 2 == 0 => { println!("Even"); },
    _ => { println!("Odd"); },
}
```

This is pretty similar to a range but more general.

### Or cases

In Rust I can collapse multiple branches together:

```rust
let description = match greeting {
    Greeting::Hello => "relatively formal",
    Greeting::Howdy | Greeting::Sup => "informal",
};
```

I can imagine extending our TypeScript `match` function to allow something similar, but I imagine the ergonomics wouldn't be great.

### Non-key-safe types

Right now we're using a [`Record`](https://www.typescriptlang.org/docs/handbook/utility-types.html#recordkeys-type) to store the association between keys and values. Unfortunately, `Record` types can only have certain types of values as keys, which leads to the `T extends string | number | symbol` bound in our code.

This means we can't write code like:

```ts
console.log(match(true, {true: "Yes", false: "No"}));
```

Well, we actually can (though TypeScript will give us an error about it), because JavaScript will automatically coerce `true` to `"true"` when using it as a key, but we certainly couldn't write:

```ts
console.log(match({"name": "Daniel"}, {{"name": "Daniel"}: true, {"name": "Dan"}: false}));
```

We could _maybe_ build up our own `Record`-like type that used a [`Map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) under the covers rather than an [`Object`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object), and which did the exhaustiveness checking that `Record` does, but that sounds like a lot of work (and we wouldn't be able to write nice literals like we have been here).

### Data-bearing variants.

One place that more complex sum types can get really powerful is the fact that they can have associated data, e.g.

```rust
enum Number {
    Zero,
    One,
    Many(u8),
}

let number = Number::Many(4);

let message = match number {
    Number::Zero => "We don't have any, sorry.",
    Number::One => "This is the last one, are you sure you need it?",
    Number::Many(n) => &format!("Of course you can have one! There will be {} left afterwards.", n-1),
};
println!("{}", message);
```

In TypeScript we often represent this as something like:

```ts
type Result =
    | {success: true, value: number }
    | {success: false, error: string}
;

let result: Result = {success: true, value: 7};
if (Math.random() < 0.1) {
    result = {success: false, error: "You got unlucky, sorry"};
}

if (result.success) {
    // TypeScript doesn't complain that result.value may not be set,
    // because we guarded the check by checking the `success` field.
    console.log(`Great! The number was ${result.value}`);
} else {
    // TypeScript doesn't complain that result.error may not be set,
    // because we guarded the check by checking the `success` field.
    console.error(`Oh no, there was an error: ${result.error}`);
}
```

Our solution here doesn't cover this kind of sum type for similar reasons to why it doesn't handle non-key-safe types. If you're interested in this kind of support you may want to check out something like [TS-Pattern](https://github.com/gvergnaud/ts-pattern).

## Conclusion

When working with TypeScript, I've definitely been missing some of the more thorough type-safety languages like Rust offer. But I've been finding it refreshing how easy it's been to build up simple equivalents that offer similar capabilities in some contexts. And flow typing is pretty fancy!

## Footnotes

[^1]: I say _basic_ sum types because in many languages sum types can include associated data. I'm intentionally ignoring that case for now, though if you're interested in it you may want to check out something like [TS-Pattern](https://github.com/gvergnaud/ts-pattern).
