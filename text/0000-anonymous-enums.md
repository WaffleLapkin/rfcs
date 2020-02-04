- Feature Name: `anonymous_enums`
- Start Date: 2020-02-04
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Add `A | B` syntax that represents anonymous enum like `enum Either<A, B> { Left(A), Right(B) }`. This feature has a lot in common with tuple syntax `(A, B)` which is anonymous strust like `struct Tuple<A, B>(A, B)`

Note: this feature was previously rejected at least 3 times:
- [First](https://github.com/rust-lang/rfcs/pull/402)
- [Second](https://github.com/rust-lang/rfcs/pull/514)
- [Third](https://github.com/rust-lang/rfcs/pull/1154)

However this RFC differs a lot from previous.

# Motivation
[motivation]: #motivation

Rust supports both kinds of composite algebraic data type: product types (structs) and sum types (enums). 
Rust has anonymous structs (tuples) but no anonymous enums, leaving a gap in it's type system.

|              |   named   | anonymous |
|-------------:|:---------:|:---------:|
|     products |  structs  |   tuples  |
|         sums |   enums   |    ???    |
| exponentials | functions |  closures |

This RFC resolves this gap.

In rust, when you need to accept or return either of some types, it's common to use enums. 
However in some cases this aproach leads to a lot of boilerplate and crates like [`either`](https://docs.rs/either) and [`frunk`](https://docs.rs/frunk/0.3.1/frunk/coproduct).
This feature could reduce this builerplate.

Also, when you have `fn f() -> impl Trait { ... }` that, depending on some condition, return either of some types, you also need to return enum. 
Which, again, leads to a lot of boilerplate (see e.g. [`futures::future::Either`](https://docs.rs/futures-util/0.3.2/src/futures_util/future/either.rs.html#11-16))

// TODO: better write motivation

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
<!--
Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
-->

## Basic concept

```rust
struct A;
struct B;

let it: A | B = ::0(A);
match it {
    ::0(a) => ...,
    ::1(b) => ...,
}
// desugars to
let it: AnonymousEnum2<A, B> = AnonymousEnum2::_0(A);
//      ^^^^^^^^^^^^^^ ---- owned by core
match it {
    AnonymousEnum2::_0(a) => ...,
    AnonymousEnum2::_1(b) => ...,
}
```

The `::n` syntax was chosen because it
- mirrors tuple's `.n` syntax
- shows that this is a type/enum variant (like `Enum::Var` but with `Enum` ommited since it's anonymous)

## Working with errors
```rust
fn read<T>() -> Result<T, io::Error | E::Err>
where
    T: FromStr,
{
    // There are 2 thing you need to note:
    //   1. We can't use `?` dicerectly, since `A | B` can't implement both `From<A>` and `From<B>` :(
    //   2. We can use `::0` as a function, just like `Either::Left`, `Enum::Var`
    let string = read_file().map_err(::0)?;
    T::from_str(&string[..]).map_err(::1)
}
```

Because `?` works badly with anonimous enums this example isn't as argonomic as it could be, 
but it's still a good alternative for using `Either` or creating new error types. (both of which are vulnerable to the same)

## More than 2 variants

This works for any number of variants:

```rust
let it: T0 | T1 | ... | Tn = ::x(Tx);
match it {
    ::0(_) => ...,
    ::1(_) => ...,
    ...
    ::n(_) => ...,
}
```

## Flattening

This RFC doesn't propose any kinds of flattening i.e. if `T` is `B | C`, then `A | T`  is not `A | B | C`:
```rust
type T = B | C | D;

let it: A | T | E = ::1(::0(B));
match it {
    ::0(a) => ...,
    ::1(t) => match t {
        ::0(b) => ...,
        ::1(c) => ...,
        ::2(d) => ...,
    }
    ::2(e) => ...,
}
// or
match it {
    ::0(a) => ...,
    ::1(::0(b)) => ...,
    ::1(::1(c)) => ...,
    ::1(::2(d)) => ...,
    ::2(e) => ...,
}
```
This code is a bit tricky, but anonymous enums aren't expected to be used with a lot of variants/big nestting, so I hope this is ok.

## Automatic Trait Implementation

Some traits like `Copy`, `Clone`, `Debug`, `Eq`, `Hash`, etc obviously should be implemented for `T0 | ... | Tx` where `T0: Trait, ..., Tx: Trait` just like for tuples and arrays.

And just like for tuples and arrays there probably should be some limit on number of types.

Trait that also could be implemented, but it's not that obvious:
- `Iterator`
- `Future`
- TODO: continue the list

Note that anonimous enums can only implement object safe traits, so it's impossible (at least resonably) implement traits like `Default` or `From`.
Also semantics of `Ord` are questionable.

Also, see [Implementing traits for `A | ... | T`][implementing-traits].

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

// TODO

<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work. -->

# Drawbacks
[drawbacks]: #drawbacks

- This RFC adds quite a complicated feature.
- Syntax is quite unintuitive (but I couldn't find better syntax that works well with `A | A` and generics)

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

// TODO

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this? -->


As said there were 3 takes to this feature before this. 
This RFC tryes to resolve problems from previous tryes, but also adds some questions.

Comprison table:

|                         problems/properties                        |          First          |            Second            |   Third  |    This    |
|:------------------------------------------------------------------:|:-----------------------:|:----------------------------:|:--------:|:----------:|
|      `A \| A` is illegal and so <br>doesn't work with generics     |            ⭕️            |               ⭕️              |     ✅    |      ✅     |
| is not the same as <br>`enum Either<A, B> { Left(A), Right(B) }`\* |            ✅            |               ⭕️              |     ✅    |      ✅     |
|                        works badly with `?`                        |            ✅            |               ✅              |     ⭕️    |      ⭕️     |
|                     pattern syntax for `A \| B`                    |         `a @ A`         | `a as A`;<br>`both as A + B` | `(a\|!)` |  `::0(a)`  |
|   Allows calls to trait methods <br>implemented by both types\*\*  |          allow          |             allow            | disallow |  disallow  |
|                Allow coersion `A` to `A \| B`\*\*\*                |          allow          |             allow            | disallow |  disallow  |
| Creation syntax for `A \| B`                                       | `A`;<br>`A as (A \| B)` | `A`;<br>`A as (A \| B)`      | `(A\|!)` | `::0(A)`   |
| Allow single variant enum                                          | disallow                | disallow                     | allow    | unresolver |
| Allow zero variant enum                                            | disallow                | disallow                     | disallow | unresolver |

(⭕️ - bad, ✅ - nice)
- `*` in second RFC `A | B` means either `A`, `B` or `(A, B)`. Thats unintutive.
- `**` in `first` and `second` RFCs, if `A: Trait` and `B: Trait`, then you can call object-safe methods on `A | B`. 
  However this brings a lot of questions like "is `(A | B): Trait`?", "exactly which traits/methods should be allowed?",
  so was removed later.
- `***` in `first` and `second` RFCs its allowed to write `let _: A | B = A;`, but there 2 problems:
  1. this means that `let x = A;` doesn't guarantee that `x` has the type `A`, because it can be later required to be `A | ...`
  2. it doesn't work when `A | A` is allowed
  

# Prior art
[prior-art]: #prior-art

// TODO

<!-- Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features. -->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

 1. Should one variant enum be allowed? `(A |) ~= enum _<A> { _0(A) }`
 2. Should empty enum be allowed? `(|) ~= ! ~= core::convert::Infallible`
 3. Some types are unique up to isomorphism, for example,`(i32, String)|(i32, f64)` and `(i32, String | f64)`. So is this code correct?
```rust
let foo: (i32, f64)|(i32, f32) = ::0(2, 6.0f64);

match foo {
    // x is i32, y is f64, z is f32
    (x, (y | z)) => ...,
}
    ```

# Future possibilities
[future-possibilities]: #future-possibilities

<!-- Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information. -->

## Anonymous variants in common enum

With this feature it's also reasonable to add anonymous variants syntax to common enums i.e.:
```rust
enum Enum(A | B); // or `(A, B)`?

let it: Enum = Enum::0(A);
match it {
    Enum::0(_) => ...,
    Enum::1(_) => ...,
}
```

## Implementing traits for `A | ... | T`
[implementing-traits]: #implementing-traits-for-a----t

It's possible to write a macro (maybe not in std) that automaticaly implements "simple" traits by signature, e.g.:

```rust
impl_enums! {
    pub trait Debug {
        fn fmt(&self, f: &mut Formatter) -> Result<(), Error>;
    }
}

// expands to

impl<A, B> Debug for A | B 
where
    A: Debug,
    B: Debug,
{
    fn fmt(&self, f: &mut Formatter) -> Result<(), Error> {
        match self {
            ::0(inner) => inner.fmt(f),
            ::1(inner) => inner.fmt(f),
        }
    }
}

...

impl<T0, T1, ..., Tx> Debug for T0 | T1 | ... | Tx 
where
    T0: Debug,
    T1: Debug,
    ...
    Tx: Debug,
{
    fn fmt(&self, f: &mut Formatter) -> Result<(), Error> {
        match self {
            ::0(inner) => inner.fmt(f),
            ::1(inner) => inner.fmt(f),
            ...
            ::x(inner) => inner.fmt(f),
        }
    }
}
```

Where `x` is some limit, like `32` for arrays or `12` for tuples.

By "simple" I mean "object safe" and without generics/associated types.

## Macros // TODO: more clear topic

Extend `macro_rules!` syntax so this would be possible:

```rust
pub trait Id {
    type Id: ?Sized;
}

impl<T: ?Sized> Id for T {
    type Id = T;
}

pub const fn type_eq::<A, B>() 
where
    A: Id<Id = B>,
{}

macro_rules! enum_to_tuple {
    ( $( $T:ty )|+ ) => ( $( $T ),+ );
}

macro_rules! tuple_to_enum {
    ( $( $T:ty ),+ ) => ( $( $T )|+ );
}

struct A; struct B; struct C;

const _: () = {
    type_eq::<enum_to_tuple!(A | B | C), (A, B, C)>();
    type_eq::<tuple_to_enum!(A, B, C), A | B | C>();
};
```
