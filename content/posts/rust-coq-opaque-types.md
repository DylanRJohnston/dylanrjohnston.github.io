---
title: "Formally Verifying Rust's Opaque Types"
date: 2022-08-01
draft: false
tags: ["rust", "type theory", "formal verification"]
cover:
  image: https://live.staticflickr.com/2069/1989724551_607a6f477b_b.jpg
---

# Introduction

The other day I was reading [this blog post](https://varkor.github.io/blog/2018/07/03/existential-types-in-rust.html) covering existential types in Rust (also known as `impl Trait` or opaque types). In that blog post, the author makes the following claim.

> We’re going to have to take a slight diversion into type theory here because it motivates a result that is perhaps intuitive. The following proposition holds in intuitionistic logic: ((∃ x. P(x)) → Q) ⇔ (∀ x. (P(x) → Q)), which means that according to the Curry–Howard Correspondence, it also holds when considering the proposition as a type.

In particular the `((∃ x. P(x)) → Q) ⇔ (∀ x. (P(x) → Q))` caught my eye. While I was working for [Originate](https://www.originate.com/) in San Francisco back in 2017 I used my 20% time project to study Benjamin C. Pierce's excellent [Software Foundations](https://softwarefoundations.cis.upenn.edu/) series which teaches how to structure and formally verify software using the Coq Proof Assistant [(yes, it's pronounced exactly how you think, 🙄)](https://www.theregister.com/2021/06/15/coq_programming_language_change/). It completely changed the way I look at programming for the better but it's not a skill that I get to practice regularly and as you get older you have to either use it or lose it. So I thought I'd try my hand at seeing if I remembered enough of Coq to be able to prove the above statement.

# Trait Dispatch in Rust

To understand why we even care about the above proposition, it's worth taking a moment to understand how `impl Trait` is used in Rust, if you're not interested in Rust and just want to get to the Coq tutorial you should skip this section and jump head to [#Coq](#coq)

In Rust, to abstract over common functionality, we have Traits (interfaces in most other languages). Below is the simple `ToString` trait from the standard library.

```rust
trait ToString {
  fn to_string(&self) -> String;
}
```

If we want to write a function that accepts things that can be turned into strings we have a few different options depending on whether we want static or dynamic dispatch. To demonstrate the available options we'll use a contrived toy function `yell` which turns the input into a string, convert it to all caps, and prints it to stdout.

## Static Dispatch

```rust
fn yell<S: ToString>(stringable: S) {
  println!(stringable.to_string().to_uppercase())
}
```

In Rust, generics are monomorphised. The compiler will create a copy and unique name for each type the function is called with, meaning each method call on the trait points to one, and only one function so the call can be statically dispatched. For example

```rust
yell(42);
yell("According to all known laws of aviation ...");
```

This will result in the creation of two functions for each call

```rust
fn yell__i32(stringable: i32) {
  println!(stringable.to_string().to_uppercase())
}

fn yell__at_str(stringable: &str) {
  println!(stringable.to_string().to_uppercase())
}
```

transforming the call site also into their respective monomorphised functions

```rust
yell__i32(42);
yell__at_str("According to all known laws of aviation ...");
```

So when we call the trait method `to_string()` in each of those monomorphised functions, the rust compiler knows precisely which function call to make and doesn't require chasing pointers (dynamic dispatch). It also means the stack layouts of calling those functions can differ but we'll get into that in a moment when talking about trait objects and object safety (spoilers!).

## Dynamic Dispatch

Another way to write our yell function is

```rust
fn yell(stringable: &dyn ToString) {
  println!(stringable.to_string().to_uppercase())
}
```

which from the name of this subsection and the use of the `dyn` keyword may have tipped you off to the fact it uses dynamic dispatch. If you're familiar with how interfaces work in Golang then you already know what's going on here.

In this case, rust doesn't copy the yell method for each type it's called with but instead `&dyn ToString` is often what's known as a "fat pointer". In that, it contains two pointers, one to the data, and one to the vtable for the methods of the trait.

```goat
                stringable
            +----------------+
     +------+  data pointer  |
     |      +----------------+
     |      | vtable pointer +-------+
     |      +----------------+       |
+----v----+                    +-----v------+
| pointer |                    | destructor |
+---------+                    +------------+
| length  |                    | size       |
+---------+                    +------------+
    &str                       | alignment  |
                               +------------+
                               | to_string()|
                               +------------+
                                   vtable
```

It's called "dynamic dispatch" because we don't know which function we're going to call until runtime when we follow the vtable pointer to find the particular `to_string()` method we're going to call. Most of the time the overhead of this pointer chasing doesn't matter and your application probably has performance bottlenecks elsewhere, but sometimes every pointer dereference counts.

## Object Saftey

Beyond questionably legitimate concerns about performance, when you stop to think about how the compiler would layout the stack to call a dynamically dispatched function you may realise a fairly large limitation on what kinds of traits can be dynamically dispatched. **They have to be the same size!** Or more precisely, the function inputs and outputs must all be of the same size!

Think about the following trait from the standard library

```rust
pub trait Clone {
    fn clone(&self) -> Self;
}
```

What is the amount of space we need to leave on the stack when calling `clone()`? Well, it returns Self, which is whatever the type that implements clone is, so i32, i64, or String all take up different amounts of space on the stack so it's impossible for us to figure out how to layout the stack at compile time when we don't know the size of the type until runtime! This is what's known in rust as **Object Saftey**. A trait that is **not** Object Safe contains methods with either its arguments or return values that do not have a constant size across all their possible implementations and therefore it is not safe to turn them into a **Trait Object** for dynamic dispatch.

In addition to trait methods that return Self, trait methods involving generic arguments or return values are also not "object safe" because the generic methods cannot be monomorphised behind a dynamic dispatch as it brushes up against the halting problem trying to figure how many entries of the monomorphised method are required in the vtable. If you tried to populate the vtable with every possible type in your application for each generic method it would grow enormous!

## `impl Trait`

So now we understand that sometimes we cannot even use dynamic dispatch even if we don't care about the overhead from pointer chasing. We're stuck with going back to our static dispatch approach of monomorphising generic functions.

```rust
fn yell<S: ToString>(stringable: S) {
  println!(stringable.to_string().to_uppercase())
}
```

The additional syntactic and mental overhead of tracking those generic parameters isn't so bad with a single generic parameter, but as the number of generic parameters grows, it can become unwieldy. For example, this function is taken from an application I'm working on that connects several different financial applications and follows the Ports and Adapters (also known as Clean or Hexagonal architecture) pattern of hiding concrete side effects behind simple interfaces and injecting them as needed.

```rust
pub fn process<E, B, S>(
    config: transformer::Config,
    expense_tracker: E,
    budget: B,
    state: S,
) -> Result<()>
where
  E: ExpenseTracker,
  B: Budget,
  S: State,
   { ... }
```

I don't know about you, but I find that kind of hard to read, especially since we're introducing generic parameters, E, B, and S and only at the end of the function definition are we giving them proper constraints. Luckily Rust has an answer! `impl Trait`. We can instead rewrite the above function as.

```rust
pub fn process(
    config: transformer::Config,
    expense_tracker: impl ExpenseTracker,
    budget: impl Budget,
    state: impl State,
) -> Result<()>
```

With this in mind you might be asking yourself why would you ever using dynamic dispatch over static. The main reason is when the dispatch is well, dynamic! Such as when you have a collection of trait objects.

```rust
fn example() {
    let string = "hello";
    let integer = 3 as i32;

    let collection: Vec<&dyn ToString> = vec![&string, &integer];
}
```

Comparatively it is not possible to create a collection using `impl Trait`, nor is it possible to express this with generic constraints because the underlying opaque but concrete types are of different sizes.

```rust
fn first() -> impl ToString {
   "According to all known laws of aviation ..."
}

fn second() -> impl ToString {
    42
}

fn example() {
    vec![first(), second()];
    //      Error ^^^
    // mismatched types
    // expected opaque type `impl ToString` (opaque type at <src/main.rs:1:15>)
    // found opaque type `impl ToString` (opaque type at <src/main.rs:4:16>)
    // distinct uses of `impl Trait` result in different opaque types
}
```

But how to understand what `impl Trait` means? One way to understand it is it's saying that there exists a type (unnamed) that implements the trait, the fact that it's unnamed is why `impl Trait` is often referred to as **opaque types**, or **existential types**. We're taking for granted that a type, some type, exists that satisfies this trait, but is it always safe to take the prior generic function and transform it into the existential form and vice versa? The introductory paragraph claims that it always is `((∃ x. P(x)) → Q) ⇔ (∀ x. (P(x) → Q))`, but let's not take some guy's word for it, we can do better! Let's prove it!

# Coq

The Coq toolchain is a bit tricky to set up locally, but luckily we don't have to as there's an excellent [online IDE](https://coq.vercel.app/scratchpad.html) that you can use instead. To translate our position into a form that Coq understands we can write

```coq
Theorem impl_trait_transform:
forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
```

First, let's take a second to understand our proposed theorem.

```coq
forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
```

Firstly `forall (Trait: Type -> Prop) (Result: Prop)` is saying that what we want to prove should be true for every possible Trait method, and any possible return value of that Trait method, we can't rely on anything true about any particular trait, our results are universal.

### split

Our proposition is then broken into two halves separated by `<->`. This is known as [material equivalence](https://en.wikipedia.org/wiki/If_and_only_if), or "if and only if", which is to say the left is true, if and only if the right is true, and visa versa. Therefore to prove it, we have to prove that the left proposition implies the right proposition, and also prove that the right proposition implies the left proposition. So we started trying to prove one thing, and now we have to prove two things! This process is very common in formal verification. To prove your goal, you have to break it apart and prove sub-sections of it. To prove the `A & B` you have to prove `A`, and you have to prove `B`, to prove `A | B` you can take your pick of `A` or `B` whichever you think is easier to prove.

So now we have to prove

```coq
- (exists t, Trait(t)) -> Result) -> (forall t, (Trait(t) -> Result)
- (forall t, (Trait(t) -> Result) -> (exists t, Trait(t)) -> Result)
```

A transformation in Coq is called a Tactic. The tactic for splitting our goal into two separate goals like this is unsurprisingly called `split`. To evaluate your proof up to where your cursor is press `CMD+Enter`. You should see the goals screen on the right update to reflect the new state of trying to prove our theorem.

```coq
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
```

Should give the output in the goals panel of

```coq
2 goals
  Trait : Type -> Prop
  Result : Prop
------------------------
(exists t : Type, Trait t) -> Result) -> forall t : Type, Trait t -> Result

subgoal 2 is:
(forall t : Type, Trait t -> Result) -> (exists t : Type, Trait t) -> Result

```

### First Goal

To focus on one goal at a time we use `-` to denote each branch of the proof. We'll use it to focus on the first part of our proof.

```coq
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
  -
```

### Tactic `intros`

```coq {linenos=table}
1 goal
Trait: Type -> Prop
Result: Prop
------------------------
((exists t : Type , Trait t)  -> Result) -> forall t : Type , Trait t -> Result
```

The way to understand trying to prove an implication like `A -> B -> C`, is that we get to assume, `A`, and `B`, when trying to prove `C`. So with the above goal, we get to assume we already have the two antecedent terms `((exists t : Type , Trait t) -> Result)` and `forall t : Type , Trait t`. The tactic that lets us do this is called `intro` for one variable at a time, or `intros` if you want to "introduce" multiple at the same time. Naming these implied assumptions is hard so feel free to name them however you want or leave it blank and let Coq auto name them.

```coq
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
  - intros existsTrait type trait.
```

### Tactic `apply`

```coq
1 goal
Trait : Type -> Prop
Result : Prop
existsTrait : (exists t : Type, Trait t) -> Result
type : Type
trait : Trait type
------------------------
Result
```

Implications work the opposite way when they're in our assumptions. Right now our goal is to make a `Result`, in our assumptions we have an implication `existsTrait : (exists t : Type, Trait t) -> Result` which can give us a `Result` if we're able to prove its antecedent. So by "applying" our assumption, we can shift our goal to trying to prove the assumptions antecedent. Maybe it's a dead end and we'll have to backtrack, but `Result` doesn't appear anywhere else in our assumptions so it's worth a try. The tactic to do this is unsurprisingly called **apply**.

```coq
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
  - intros existsTrait type trait.
    apply existsTrait.
```

### Tactic `exists`

```coq
1 goal
Trait : Type -> Prop
Result : Prop
existsTrait : (exists t : Type, Trait t) -> Result
type : Type
trait : Trait type
------------------------
exists t : Type, Trait t
```

Now our goal is `exists t : Type, Trait t`, much like `forall` and `intros`, there exists a tactic for dealing with existentially quantified variables, called `exists`. Although unlike intros, which give us new assumptions, `exists` demands that we prove that a term exists. It is in some sense the opposite of `forall/intros`, it consumes a term rather than giving us one. Luckily, after examining our assumptions, we can see that our previous `intros` gave us an assumption of kind `type : Type` that matches the kind demanded by the "exists". So we can get rid of it with `exists`.

```coq
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
  - intros existsTrait type trait.
    apply existsTrait.
    exists type.
```

### Tactic `assumption`

```coq
1 goal
Trait : Type -> Prop
Result : Prop
exists : Trait(exists t : Type, Trait t) -> Result
type : Type
trait : Trait type
------------------------
Trait type
```

Now you may notice that our goal exactly matches one of our assumptions. So we can use the tactic `assumption` to finish this branch of the proof! One down and one to go!

```
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
  - intros existsTrait type trait.
    apply existsTrait.
    exists type.
    assumption.
  -
```

### Second Goal

```
1 goal
Trait : Type -> Prop
Result : Prop
------------------------
(forall t : Type, Trait t -> Result) -> (exists t : Type, Trait t) -> Result
```

Now we're proving that the material equivalence holds in the other direction, and just like the first branch we use `intros` to move the left-hand side of the implications into our assumptions.

```coq
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
  - intros existsTrait type concreteTrait.
    apply existsTrait.
    exists type.
    assumption.
  - intros anyTrait existsTrait.
```

### Tactic `destruct`

```coq
1 goal
Trait : Type -> Prop
Result : Prop
anyTrait : forall t : Type , Trait t  -> Result
existsTrait : exists t : Type, Trait t
------------------------
Result
```

At this point you might be tempted to try and apply `anyTrait` just like we did for our first goal, trying to switch the goal from result to `forall t : Type , Trait t`, however, this doesn't work with Coq complaining that `Unable to find an instance for the variable t`. This is because t is still universally quantified right now, whereas in our first goal t was existentially quantified. So we're going to need to find ourselves a `t: Type` before we can shift our goal.

We also have the `existsTrait: exists t : Type, Trait t` assumption. Luckily, when we have a term like this, we can ask Coq if this expression implies anything else that is not currently in our assumptions. We do this with the `inversion` or `destruct` tactics. Coq will auto-name the new assumptions but we can also give them meaningful names with `as`.

```coq
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
  - intros existsTrait type concreteTrait.
    apply existsTrait.
    exists type.
    assumption.
  - intros anyTrait existsTrait.
    destruct existsTrait as (t, trait).
```

### Tactic `apply _ with _`

```coq
Trait : Type -> Prop
Result : Prop
anyTrait : forall t : Type, Trait t -> Result
t : Type
trait : Trait type
------------------------
Result
```

Coq has broken apart `existsTrait` into its two assumptions. So now we have an assumption of kind `Type`! We can now solve the problem we had before about applying `anyTrait`.

```coq
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
  - intros existsTrait type concreteTrait.
    apply existsTrait.
    exists type.
    assumption.
  - intros anyTrait existsTrait.
    destruct existsTrait as (t, trait).
    apply anyTrait with t.
```

### Tactic `assumption`

```coq
Trait : Type -> Prop
Result : Prop
anyTrait : forall t : Type, Trait t -> Result
t : Type
trait : Trait t
------------------------
Trait t
```

Now just like before, our goal matches one of our assumptions, so we can finish up this branch with `assumption`.

```coq
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
  - intros existsTrait type concreteTrait.
    apply existsTrait.
    exists type.
    assumption.
  - intros anyTrait existsTrait.
    destruct existsTrait as (t, trait).
    apply anyTrait with t.
    assumption.
```

### QED

```coq
No more goals.
```

Now that we're proved all of our goals, we get to do the best thing about formal verification. The sweet three-letter acronym. **QED**.

```coq
Theorem impl_trait_transform: forall (Trait: Type -> Prop) (Result: Prop),
  ((exists t, Trait(t)) -> Result) <-> (forall t, (Trait(t) -> Result)).
Proof.
  split.
  - intros existsTrait type concreteTrait.
    apply existsTrait.
    exists type.
    assumption.
  - intros anyTrait existsTrait.
    destruct existsTrait as (t, trait).
    apply anyTrait with t.
    assumption.
Qed.
```

# Conclusion

And there you go, we've formally verified that our transformation from generic arguments with trait bounds into existential `impl Trait` arguments is always valid. Of course, Rust would not have implemented the feature had it not made sense, but I think it's gratifying to be able to prove such a cryptic statement like `((∃ x. P(x)) → Q) ⇔ (∀ x. (P(x) → Q))`

Hopefully, this has given you a bit of a taste of how to reason when it comes to formally verify programs which is such an alien style of programming. I've always likened it to playing with Legos where the bricks are parts of your program. You know the final shape you want, you just have to keep exploring how they combine to get there.

If you'd like to explore more formal verification I'd highly recommend checking out [Software Foundations](https://softwarefoundations.cis.upenn.edu/) or Edwin Brady's [Type-Driven Development with Idris](https://www.manning.com/books/type-driven-development-with-idris).
