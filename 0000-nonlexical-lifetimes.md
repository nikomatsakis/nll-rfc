- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Extend Rust's borrow system to support **non-lexical lifetimes** --
these are lifetimes that are based on the control-flow graph, rather
than lexical scopes. The RFC describes in detail how to infer these
new, more flexible regions, and also describes how to adjust our error
messages.

# Motivation
[motivation]: #motivation

## What is a lifetime?

The basic idea of the borrow checker is that values may not be mutated
or moved while they are borrowed. But how do we know whether a value
is borrowed? The idea is quite simple: whenever you create a borrow,
the compiler assigns the resulting reference a **lifetime**. This
lifetime corresponds to the span of the code where the reference may
be used. The compiler will infer this lifetime to be the smallest
lifetime that it can that still encompasses all the uses of the
reference.

Note that Rust uses the term lifetime in a very particular way.  In
everyday speech, the word lifetime can be used in two distinct -- but
similar -- ways:

1. The lifetime of a **reference**, corresponding to the span of time in
   which that reference is **used**.
2. The lifetime of a **value**, corresponding to the span of time
   before that value gets **freed** (or, put another way, before the
   destructor for the value runs).

This second span of time, which describes how long a value is valid,
is of course very important. To distinguish the two, we refer to that
second span of time as the value's **scope**. Naturally, lifetimes and
scopes are linked to one another. Specifically, if you make a
reference to a value, the lifetime of that reference cannot outlive
the scope of that value. Otherwise, your reference would be pointing
into freed memory.

To better see the distinction between lifetime and scope, let's
consider a simple example. In this example, the vector `data` is
borrowed (mutably) and the resulting reference is passed to a function
`capitalize`. Since `capitalize` does not return the reference back,
the *lifetime* of this borrow will be confined to just that call. The
*scope* of data, in contrast, is much larger, and corresponds to a
suffix of the fn body, stretching from the `let` until the end of the
enclosing scope.

```rust
fn foo() {
    let mut data = vec!['a', 'b', 'c']; // --+ 'scope
    capitalize(&mut data[..]);          //   |
//  ^~~~~~~~~~~~~~~~~~~~~~~~~ 'lifetime //   |
    data.push('d');                     //   |
    data.push('e');                     //   |
    data.push('f');                     //   |
} // <---------------------------------------+

fn capitalize(data: &mut [char]) {
    // do something
}
```

This example also demonstrates something else. Lifetimes in Rust today
are quite a bit more flexible than scopes (if not as flexible as we
might like, hence this RFC):

- A scope generally corresponds to some block (or, more specifically,
  a *suffix* of a block that stretches from the `let` until the end of
  the enclosing block) \[[1](#temporaries)\].
- A lifetime, in contrast, can also span an individual expression, as
  this example demonstrates. The lifetime of the borrow in the example
  is confined to just the call to `capitalize`, and doesn't extend
  into the rest of the block. This is why the calls to `data.push`
  that come below are legal.

So long as a reference is only used within one statement, today's
lifetimes are typically adequate. Problems arise however when you have
a reference that spans multiple statements. In that case, the compiler
requires the lifetime to be the innermost expression (which is often a
block) that encloses both statements, and that is typically much
bigger than is really necessary or desired. Let's look at some example
problem cases. Later on, we'll see how non-lexical lifetimes fix these
cases.

## Problem case #1: references assigned into a variable

One common problem case is when a reference is assigned into a
variable. Consider this trivial variation of the previous example,
where the `&mut data[..]` slice is not passed directly to
`capitalize`, but is instead stored into a local variable:

```rust
fn bar() {
    let mut data = vec!['a', 'b', 'c'];
    let slice = &mut data[..]; // <-+ 'lifetime
    capitalize(slice);         //   |
    data.push('d'); // ERROR!  //   |
    data.push('e'); // ERROR!  //   |
    data.push('f'); // ERROR!  //   |
} // <------------------------------+
```

The way that the compiler currently works, assigning a reference into
a variable means that its lifetime must be as large as the entire
scope of that variable. In this case, that means the lifetime is now
extended all the way until the end of the block. This in turn means
that the calls to `data.push` are now in error, because they occur
during the lifetime of `slice`. It's logical, but it's annoying.

In this particular case, you could resolve the problem by putting
`slice` into its own block:

```rust
fn bar() {
    let mut data = vec!['a', 'b', 'c'];
    {
        let slice = &mut data[..]; // <-+ 'lifetime
        capitalize(slice);         //   |
    } // <------------------------------+
    data.push('d'); // OK
    data.push('e'); // OK
    data.push('f'); // OK
}
```

Since we introduced a new block, the scope of `slice` is now smaller,
and hence the resulting lifetime is smaller. Of course, introducing a
block like this is kind of artificial and also not an entirely obvious
solution.

## Problem case #2: conditional control flow

Another common problem case is when references are used in only match
arm (or, more generally, one control-flow path). This most commonly
arises around maps. Consider this function, which, given some `key`,
processes the value found in `map[key]` if it exists, or else inserts
a default value:

```rust
fn process_or_default() {
    let mut map = ...;
    let key = ...;
    match map.get_mut(&key) { // -------------+ 'lifetime
        Some(value) => process(value),     // |
        None => {                          // |
            map.insert(key, V::default()); // |
            //  ^~~~~~ ERROR.              // |
        }                                  // |
    } // <------------------------------------+
}
```

This code will not compile today. The reason is that the `map` is
borrowed as part of the call to `get_mut`, and that borrow must
encompass not only the call to `get_mut`, but also the `Some` branch
of the match. The innermost expression that encloses both of these
expressions is the match itself (as depicted above), and hence the
borrow is considered to extend until the end of the
match. Unfortunately, the match encloses not only the `Some` branch,
but also the `None` branch, and hence when we go to insert into the
map in the `None` branch, we get an error that the `map` is still
borrowed.

This *particular* example is relatively easy to workaround. One can
(frequently) move the code for `None` out from the `match` like so:

```rust
fn process_or_default1() {
    let mut map = ...;
    let key = ...;
    match map.get_mut(&key) { // -------------+ 'lifetime
        Some(value) => {                   // |
            process(value);                // |
            return;                        // |
        }                                  // |
        None => {                          // |
        }                                  // |
    } // <------------------------------------+
    map.insert(key, V::default());
}
```

When the code is adjusted this way, the call to `map.insert` is not
part of the match, and hence it is not part of the borrow.  While this
works, it is of course unfortunate to require these sorts of
manipulations, just as it was when we introduced an artificial block
in the previous example.

## Problem case #3: conditional control flow across functions

While we were able to work around problem case #2 in a relatively
simple, if irritating, fashion. there are other variations of
conditional control flow that cannot be so easily resolved. This is
particularly true when you are returning a reference out of a
function. Consider the following function, which returns the value for
a key if it exists, and inserts a new value otherwise (for the
purposes of this section, assume that the `entry` API for maps does
not exist):

```rust
fn get_default<'m,K,V:Default>(map: &'m mut HashMap<K,V>,
                               key: K)
                               -> &'m mut V {
    match map.get_mut(&key) { // -------------+ 'm
        Some(value) => value,              // |
        None => {                          // |
            map.insert(key, V::default()); // |
            //  ^~~~~~ ERROR               // |
            map.get_mut(&key).unwrap()     // |
        }                                  // |
    }                                      // |
}                                          // v
```

At first glance, this code appears quite similar the code we saw
before. And indeed, just as before, it will not compile. But in fact
the lifetimes at play are quite different. The reason is that, in the
`Some` branch, the value is being **returned out** to the caller.
Since `value` is a reference into the map, this implies that the `map`
will remain borrowed **until some point in the caller** (the point
`'m`, to be exact). To get a better intuition for what this lifetime
parameter `'m` represents, consider some hypothetical caller of
`get_default`: the lifetime `'m` then represents the span of code in
which that caller will use the resulting reference:

```rust
fn caller() {
    let mut map = HashMap::new();
    ...
    {
        let v = get_default(&mut map, key); // -+ 'm
          // +-- get_default() -----------+ //  |
          // | match map.get_mut(&key) {  | //  |
          // |   Some(value) => value,    | //  |
          // |   None => {                | //  |
          // |     ..                     | //  |
          // |   }                        | //  |
          // +----------------------------+ //  |
        process(v);                         //  |
    } // <--------------------------------------+
    ...
}
```

If we attempt the same workaround for this case that we tried
in the previous example, we will find that it does not work:

```rust
fn get_default1<'m,K,V:Default>(map: &'m mut HashMap<K,V>,
                                key: K)
                                -> &'m mut V {
    match map.get_mut(&key) { // -------------+ 'm
        Some(value) => return value,       // |
        None => { }                        // |
    }                                      // |
    map.insert(key, V::default());         // |
    //  ^~~~~~ ERROR (still)                  |
    map.get_mut(&key).unwrap()             // |
}                                          // v
```

Whereas before the lifetime of `value` was confined to the match, this
new lifetime extends out into the caller, and therefore the borrow
does not end just because we exited the match. Hence it is still in
scope when we attempt to call `insert` after the match.

The workaround for this problem is a bit more involved. It relies on
the fact that the borrow checker uses the precise control-flow of the
function to determine what borrows are in scope.

```rust
fn get_default2<'m,K,V:Default>(map: &'m mut HashMap<K,V>,
                                key: K)
                                -> &'m mut V {
    if map.contains(&key) {
    // ^~~~~~~~~~~~~~~~~~ 'n
        return match map.get_mut(&key) { // + 'm
            Some(value) => value,        // |
            None => unreachable!()       // |
        };                               // v
    }

    // At this point, `map.get_mut` was never
    // called! (As opposed to having been called,
    // but its result no longer being in use.)
    map.insert(key, V::default()); // OK now.
    map.get_mut(&key).unwrap()
}
```

What has changed here is that we moved the call to `map.get_mut`
inside of an `if`, and we have set things up so that the if body
unconditionally returns. What this means is that a borrow begins at
the point of `get_mut`, and that borrow lasts until the point `'m` in
the caller, but the borrow checker can see that this borrow *will not
have even started* outside of the `if`. So it does not consider the
borrow in scope at the point where we call `map.insert`.

This workaround is more troublesome than the others, because the
resulting code is actually less efficient at runtime, since it must do
multiple lookups.

It's worth noting that Rust's hashmaps include an `entry` API that
one could use to implement this function today. The resulting code is
both nicer to read and more efficient even than the original version,
since it avoids extra lookups on the "not present" path as well:

```rust
fn get_default3<'m,K,V:Default>(map: &'m mut HashMap<K,V>,
                                key: K)
                                -> &'m mut V {
    map.entry(key)
       .or_insert_with(|| V::default())
}
```

Regardless, the problem exists for other data structures besides
`HashMap`, so it would be nice if the original code passed the borrow
checker, even if in practice using the `entry` API would be
preferable. (Interestingly, the limitation of the borrow checker here
was one of the motivations for developing the `entry` API in the first
place!)

## The rough outline of our solution

This RFC proposes a more flexible model for lifetimes. Whereas
previously lifetimes were based on the abstract syntax tree, we now
propose lifetimes that are defined via the control-flow graph. More
specifically, lifetimes will be derived based on the MIR
representation used internally in the compiler.

Intuitively, in the new proposal, the lifetime of a reference lasts
only for those portions of the function in which the reference may
later be used (where the reference is **live**, in compiler
speak). This can range from a few sequential statements (as in problem
case #1) to something more complex, such as covering one arm in a
match but not the others (problem case #2).

However, in order to sucessfully type the full range of examples that
we would like, we have to go a bit further than just changing
lifetimes to a portion of the control-flow graph. **We also have to
take location into account when doing subtyping checks**. This is in
contrast to how the compiler works today, where subtyping relations
are "absolute". That is, in the current compiler, the type `&'a ()` is
a subtype of the type `&'b ()` whenever `'a` outlives `'b` (`'a: 'b`),
which means that `'a` corresponds to a bigger portion of the function.
Under this proposal, subtyping can instead be established **at a
particular point P**. In that case, the lifetime `'a` must only
outlive those portions of `'b` that are reachable from P.

The ideas in this RFC have been implemented in
[prototype form][proto]. This prototype includes a simplified
control-flow graph that allows one to create the various kinds of
region constraints that can arise and implements the region inference
algorithm which then solves those constraints.

[proto]: https://github.com/nikomatsakis/nll

# Detailed design
[design]: #detailed-design

## Layering the design

We describe the design in "layers":

1. Initially, we will describe a based design focused on control-flow within
   one function.
2. Next, we extend the design to handle dropck, and specifically the
   `#[may_dangle]` attribute introduced by RFC 1327.
3. Finally, we will extend the design to consider named lifetime parameters,
   like those in problem case 3.

## Layer 1: Control-flow within a function

### Running Example

We will explain the design with reference to a running example, called
**Example 4**. After presenting the design, we will apply it to the three
problem cases, as well as a number of other interesting examples.

```rust
let mut foo: T = ...;
let mut bar: T = ...;
let p: &T;

p = &foo;
// (0)
if condition {
    print(*p);
    // (1)
    p = &bar;
    // (2)
}
// (3)
print(*p);
// (4)
```

The key point of this example is that the variable `foo` should only
be considered borrowed at points 0, 3, and 4, but not point 1. `bar`,
in contrast, should be considered borrowed at points 2 and 3. Neither
them need to be considered borrowed at point 4, as the reference `p`
is not used there.

We can concert this example into the control-flow graph that follows.
Recall that a control-flow graph in MIR consists of basic blocks
containing a list of discrete statements and a trailing terminator:

```
// let mut foo: i32;
// let mut bar: i32;
// let p: &i32;

A
[ p = &foo     ]
[ if condition ] ----\ (true)
       |             |
       |     B       v
       |     [ print(*p)     ]
       |     [ ...           ]
       |     [ p = &bar      ]
       |     [ ...           ]
       |     [ goto C        ]
       |             |
       +-------------/
       |
C      v
[ print(*p)    ]
[ return       ]
```

We will use a notation like `Block/Index` to refer to a specific
statement or terminate in the control-flow graph. So `A/0` and `B/4`
refer to `p = &foo` and `goto C`, respectively.

### What is a lifetime and how does it interact with the borrow checker

To start with, we will consider lifetimes as a **set of points in the
control-flow graph**; later in the RFC we will extend the domain of
these sets to include "skolemized" lifetimes, which correspond to
named lifetime parameters declared on a function. If a lifetime
contains the point P, that implies that references with that lifetime
are valid on entry to P. Lifetimes appear in various places in the MIR
representation:

- The types of variables (and temporaries, etc) may contain lifetimes.
- Every borrow expression has a desigated lifetime.

We can extend our example 4 to include explicit lifetime names. There
are three lifetimes that result. We will call them `'p`, `'foo`, and
`'bar`:

```rust
let mut foo: T = ...;
let mut bar: T = ...;
let p: &'p T;
//      --
p = &'foo foo;
//   ----
if condition {
    print(*p);
    p = &'bar bar;
    //   ----
}
print(*p);
```

As you can see, the lifetime `'p` is part of the type of the variable
`p`. It indicates the portions of the control-flow graph where `p` can
safely be dereferenced. The lifetimes `'foo` and `'bar` are different:
they refer to the lifetimes for which `foo` and `bar` are borrowed,
respectively.

Lifetimes attached to a borrow expression, like `'foo` and `'bar`, are
important to the borrow checker. Those correspond to the portions of
the control-flow graph in which the borrow checker will enforce its
restrictions. In this case, since both borrows are shared borrows
(`&`), the borrow checker will prevent `foo` from being modified
during `'foo` and it will prevent `bar` from being modified during
`'bar`. If these had been mutable borrows (`&mut`), the borrow checker
would have prevented **all** access to `foo` and `bar` during those
lifetimes.

There are many valid choices one could make for `'foo` and `'bar`.
This RFC however describes an inference algorithm that aims to pick
the **minimal** lifetimes for each borrow which could possibly work.
This corresponds to imposing the fewest restrictions we can.

In the case of example 4, therefore, we wish our algorithm to compute
that `'foo` is `{A/1, B/0, C/0}`, which notably excludes the points B/1
through B/4. `'bar` should be inferred to the set `{B/3, B/4,
C/0}`. The lifetime `'p` will be the union of `'foo` and `'bar`, since
it contains all the points where the variable `p` is valid.

### Lifetime inference constraints

The inference algorithm works by analyzing the MIR and creating a
series of **constraints**. These constraints obey the following
grammar:

```
// A constraint set C:
C = true
  | C, (L1: L2) @ P    // Lifetime L1 outlives Lifetime L2 at point P

// A lifetime L:
L = 'a
  | {P}
```

Here the terminal `P` represents a point in the control-flow graph,
and the notation `'a` refers to some named lifetime inference variable
(e.g., `'p`, `'foo` or `'bar`).

Once the constraints are created, the **inference algorithm** solves
the constraints. This is done via a simple fixed-point iteration: each
lifetime variable begins as an empty set and we iterate over the
constaints, repeatedly growing the lifetimes until they are big enough
to satisfy all constraints.

(If you'd like to compare this to the prototype code, the file
[`regionck.rs`] is responsible for creating the constraints, and
[`infer.rs`] is responsible for solving them.)

[`regionck.rs`]: XXX
[`infer.rs`]: XXX

### Liveness

One key ingredient to understanding how NLL should work is
understanding **liveness**. The term "liveness" derives from compiler
analysis, but it's fairly intuitive. We say that **a variable is live
if the current value that it holds may be used later**. This is very
important to Example 4:

```rust
let mut foo: T = ...;
let mut bar: T = ...;
let p: &'p T = &foo;
// `p` is live here: its value may be used on the next line.
if condition {
    // `p` is live here: its value will be used on the next line.
    print(*p);
    // `p` is DEAD here: its value will not be used.
    p = &bar;
    // `p` is live here: its value will be used later.
}
// `p` is live here: its value may be used on the next line.
print(*p);
// `p` is DEAD here: its value will not be used.
```

Here you see a variable `p` that is assigned in the beginning of the
program, and then maybe re-assigned during the `if`. The key point is
that `p` becomes **dead** (not live) in the span before it is
reassigned.  This is true even though the variable `p` will be used
again, because the **value** that is in `p` will not be used.

Traditional compiler compute liveness based on variables. But we wish
to compute liveness for **lifetimes**. We can extend a variable-based
analysis to lifetimes by saying that a lifetime L is live at a point P
if there is some variable `p` which is live at P, and L appears in the
type of `p`. (Later on, when we cover the dropck, we will use a more
selection notion of liveness for lifetimes in which *some* of the
lifetimes in a variable's type may be live while others are not.) So,
in our running example, the lifetime `'p` would be live at precisely
the same points that `p` is live. The lifetimes `'foo` and `'bar` are
never live, since they do not appear in the types of any variables.

#### Liveness-based constraints for lifetimes

The first set of constraints that we generate are derived from
liveness. Specifically, if a lifetime L is live at the point P,
then we will introduce a constraint like:

    (L: {P}) @ P

(As we'll see later when we cover solving constraints, this constraint
effectively just inserts `P` into the set for `L`. And in fact the
prototype doesn't bother to materialize such constraints, instead just
immediately inserting `P` into `L`.)

For our running example, this means that we would introduce the following
liveness constraints:

    ('p: {A/1}) @ A/1
    ('p: {B/0}) @ B/0
    ('p: {B/3}) @ B/3
    ('p: {B/4}) @ B/4
    ('p: {C/0}) @ C/0

### Subtyping

Whenever references are copied from one location to another, the Rust
subtyping rules requires that the lifetime of the source reference
**outlives** the lifetime of the target location. As discussed
earlier, in this RFC, we extend the notion of subtyping to be
**location-aware**, meaning that we take into account the point where
the value is being copied.

For example, at the point A/0, our running example contains a borrow
expression `p = &'foo foo`. In this case, the borrow expression will
produce a reference of type `&'foo T`, where `T` is the type of
`foo`. This value is then assigned to `p`, which has the type `&'p T`.
Therefore, we wish to require that `&'foo T` be a subtype of `&'p T`.
Moreover, this relation needs to hold at the point A/1 -- the
**successor** of the point A/0 where the assignment occurs (this is
because the new value of `p` is first visible in A/1). We write that
subtyping constraint as follows:

    (&'foo T <: &'p T) @ A/1
    
The standard Rust subtyping rules (two examples of which are given
below) can then "breakdown" this subtyping rule into the lifetime
constraints we need for inference:

    (T_a <: T_b) @ P
    ('a: 'b) @ P      // <-- a constraint for our inference algorithm
    ------------------------
    (&'a T_a <: &'b T_b) @ P

    (T_a <: T_b) @ P
    (T_b <: T_a) @ P  // (&mut T is invariant)
    ('a: 'b) @ P      // <-- another constraint
    ------------------------
    (&'a mut T_a <: &'b mut T_b) @ P

In the case of our running example, we generate the following subtyping
constraints:

    (&'foo T <: &'p T) @ A/1
    (&'bar T <: &'p T) @ B/3

These can be converted into the following lifetime constraints:

    ('foo: 'p) @ A/1
    ('bar: 'p) @ B/3
    
### Solving constraints

Once the constraints are created, the **inference algorithm** solves
the constraints. This is done via a simple fixed-point iteration: each
lifetime variable begins as an empty set and we iterate over the
constaints, repeatedly growing the lifetimes until they are big enough
to satisfy all constraints.

The meaning of a constraint like `('a: 'b) @ P` is that, starting from
the point P, the lifetime `'a` must include all points that are
reachable without leaving the lifetime `'b`. The implementation
[does a depth-first search starting from P][dfs]; the search stops if
we exit the lifetime `'b`. Otherwise, for each point we find, we add
it to `'a`.

In our example, the full set of constraints is:

    ('foo: 'p) @ A/1
    ('bar: 'p) @ B/3
    ('p: {A/1}) @ A/1
    ('p: {B/0}) @ B/0
    ('p: {B/3}) @ B/3
    ('p: {B/4}) @ B/4
    ('p: {C/0}) @ C/0
    
Solving these constraints results in the following lifetimes,
which are precisely the answers we expected:

    'p   = {A/1, B/0, B/3, B/4, C/0}
    'foo = {A/1, B/0, C/0}
    'bar = {B/3, B/4, C/0}

[dfs]: XXX

### Intuition for why this algorithm is correct

For the algorithm to be correct, there is a critical invariant that we
must maintain. Consider some path H that is borrowed with lifetime L
at a point P to create a reference R; this reference R (or some
copy/move of it) is then later dereferenced at some point Q.

We must ensure that the reference has not been invalidated: this means
that the memory which was borrowed must not have been freed by the
time we reach Q. If the reference R is a shared reference (`&T`), then
the memory must also not have been written (modulo `UnsafeCell`). If
the reference R is a mutable reference (`&mut T`), then the memory
must not have been accessed at all, except through the reference R.
**To guarantee these properties, we must prevent actions that might
affect the borrowed memory for all of the points between P (the
borrow) and Q (the use).**

This means that L must at least include all the points between P and
Q. There are two cases to consider. First, the case where the access
at point Q occurs through the same reference R that was created by
the borrow:

    R = &H; // point P
    ...
    use(R); // point Q

In this case, the variable R will be **live** on all the points
between P and Q. The liveness-based rules suffice for this case:
specifically, because the type of R includes the lifetime L, we know
that L must include all the points between P and Q, since R is live
there.

The second case is when the memory referenced by R is accessed, but
through an alias (or move):

    R = &H;  // point P
    R2 = R;  // last use of R, point A
    ...
    use(R2); // point Q
    
In this case, the liveness rules along do not suffice. The problem is
that the `R2 = R` assignment may well be the last use of R, and so the
**variable** R is dead at this point. However, the *value* in R will
still be dereferenced later (through R2), and hence we want the
lifetime L to include those points. This is where the **subtyping
constraints** come into play: the type of R2 includes a lifetime L2,
and the assignment `R2 = R` will establish an outlives constraint `(L:
L2) @ A` between L and L2. Moreover, this new variable R2 must be
live between the assignment and the ultimate use (that is, along the
path A...Q). Putting these two facts together, we see that L will
ultimately include the points from P to A (because of the liveness of
R) and the points from A to Q (because the subtyping requirement
propagates the liveness of R2).

Note that it is possible for these lifetimes to have gaps. This can occur
when the same variable is used and overwritten multiple times:

    let R: &L i32;
    let R2: &L2 i32;

    R = &H1; // point P1
    R2 = R;  // point A1
    use(R2); // point Q1
    ...
    R2 = &H2; // point P2
    use(R2);  // point Q2
  
In this example, the liveness constraints on R2 will ensure that L2
(the lifetime in its type) includes Q1 and Q2 (because R2 is live at
those two points), but not the "..." nor the points P1 or P2. Note
that the subtyping relationship (`(L: L2) @ A1)`) at A1 here ensures
that L also includes Q1, but doesn't require that L includes Q2 (even
though L2 has point Q2). This is because the value in R2 at Q2 cannot
have come from the assignment at A1; if it could have done, then
either R2 would have to be live between A1 and Q2 or else there would
be a subtyping constraint.

### Other examples

Let us work through some more examples. We begin with problem cases #1
and #2 (problem case #3 will be covered after we cover named lifetimes
in a later section).

#### Problem case #1.

Translated into MIR, the example will look roughly as follows:

```rust
let mut data: Vec<i32>;
let slice: &'slice mut i32;
START {
    vec = ...;
    slice = &'borrow mut data;
    capitalize(slice);
    data.push('d');
    data.push('e');
    data.push('f');
}
```

The constraints generated will be as follows:

    ('slice: {START/2}) @ START/2
    ('borrow: 'slice) @ START/2

Both `'slice` and `'borrow` will therefore be inferred to START/2, and
hence the accesses to `data` in START/3 and the following statements
are permitted.

#### Problem case #2.

Translated into MIR, the example will look roughly as follows (some
irrelevant details are elided). Note that the `match` statement is
translated into a SWITCH, which tests the variant, and a "downcast",
which lets us extract the contents out from the `Some` variant (this
operation is specific to MIR and has no Rust equivalent, other than as
part of a match).

```
let map: HashMap<K,V>;
let key: K;
let tmp0: &'tmp0 mut HashMap<K,V>;
let tmp1: &K;
let tmp2: Option<&'tmp2 mut V>;
let value: &'value mut V;

START {
/*0*/ map = ...;
/*1*/ key = ...;
/*2*/ tmp0 = &'map mut map;
/*3*/ tmp1 = &key;
/*4*/ tmp2 = HashMap::get_mut(tmp0, tmp1);
/*5*/ SWITCH tmp2 { None => NONE, Some => SOME }
}

NONE {
/*0*/ ...
/*1*/ goto EXIT;
}

SOME {
/*0*/ value = tmp2.downcast<Some>.0;
/*1*/ process(value);
/*2*/ goto EXIT;
}

EXIT {
}
```

The following liveness constraints are generated:

    ('tmp0: {START/3}) @ START/3
    ('tmp0: {START/4}) @ START/4
    ('tmp2: {SOME/0}) @ SOME/0
    ('value: {SOME/1}) @ SOME/1
    
The following subtyping-based constraints are generated:

    ('map: 'tmp0) @ START/3
    ('tmp0: 'tmp2) @ START/5
    ('tmp2: 'value) @ SOME/1

Ultimately, the lifetime we are most interested in is `'map`,
which indicates the duration for which `map` is borrowed. If we solve
the constraints above, we will get:

    'map == {START/3, START/4, SOME/0, SOME/1}
    'tmp0 == {START/3, START/4, SOME/0, SOME/1}
    'tmp2 == {SOME/0, SOME/1}
    'value == {SOME/1}

These results indicate that `map` **can** be mutated in the `None`
arm; `map` could also be mutated in the `Some` arm, but only after
`process()` is called (i.e., starting at SOME/2). This is the desired
result.

#### Example 4, invariant

It's worth looking at a variant of our running example ("Example 4").
This is the same pattern as before, but instead of using `&'a T`
references, we use `Foo<'a>` references, which are **invariant** with
respect to `'a`.  This means that the `'a` lifetime in a `Foo<'a>`
value cannot be approximated (i.e., you can't make it shorter, as you
can with a normal reference). Usually invariance arises because of
mutability (e.g., `Foo<'a>` might have a field of type `Cell<&'a
()>`). The key point here is that invariance actually makes **no
difference at all** the outcome. This is true because of
location-based subtyping.

```rust
let mut foo: T = ...;
let mut bar: T = ...;
let p: Foo<'a>;

p = Foo::new(&foo);
if condition {
    print(*p);
    p = Foo::new(&bar);
}
print(*p);
```

Effectively, we wind up with the same constraints as before, but where
we only had `'foo: 'p`/`'bar: 'p` constraints before (due to subtyping), we now
also have `'p: 'foo` and `'p: 'bar` constraints:

    ('foo: 'p) @ A/1
    ('p: 'foo) @ A/1
    ('bar: 'p) @ B/3
    ('p: 'bar) @ B/3
    ('p: {A/1}) @ A/1
    ('p: {B/0}) @ B/0
    ('p: {B/3}) @ B/3
    ('p: {B/4}) @ B/4
    ('p: {C/0}) @ C/0

The key point is that the new constraints don't affect the final answer:
the new constraints were already satisfied with the older answer.

#### vec-push-ref

In previous iterations of this proposal, the location-aware subtyping
rules were replaced with transformations such as SSA form. The
vec-push-ref example demonstrates the value of location-aware
subtyping in contrast to these approaches.

```
let foo: i32;
let vec: Vec<&'vec i32>;
let p: &'p i32;

foo = ...;
vec = Vec:new();
p = &'foo foo;
if true {
    vec.push(p);
} else {
    // Key point: `foo` not borrowed here.
    use(vec);
}
```

This can be converted to control-flow graph form:

```
block START {
    v = Vec::new();
    p = &'foo foo;
    goto B C;
}
            
block B {
    vec.push(p);
    goto EXIT;
}
                    
block C {    
    // Key point: `foo` not borrowed here
    goto EXIT;
}
                                
block EXIT {
    use(vec);
}
```

Here the relations from liveness are:

    ('vec: {START/1}) @ START/1
    ('vec: {START/2}) @ START/2
    ('vec: {B/0}) @ B/0
    ('vec: {C/0}) @ C/0
    ('p: {START/2}) @ START/2
    ('p: {B/0}) @ B/0

Meanwhile, the call to `vec.push(p)` establishes this subtyping
relation:

    ('p: 'vec) @ B/1
    ('foo: 'p) @ START/2

The solution is:

    'vec = {START/1, START/2, B/0, C/0}
    'p = {START/2, B/0}
    'foo = {START/2, B/0}

What makes this example interesting is that **the lifetime `'vec` must
include both halves of the `if`** -- because it is used after the `if`
-- but `'vec` only becomes "entangled" with the lifetime `'p` on one
path. Thus even though `'vec` has to outlive `'p`, `'p` never winds up
including the "else" branch thanks to location-aware subtyping.

## Layer 2: Accomodating dropck

MIR includes an action that corresponds to "dropping" a value:

    DROP(lvalue)

This operation executes the destructor for `lvalue`, effectively
"de-initializing" the memory in which the value resides.

Interestingly, dropping a value frequently does not require that the
lifetimes in the dropped value be valid. After all, dropping a
reference of type `&'a T` or `&'a mut T` is defined as a no-op, so it
does not matter if the reference points at valid memory. In cases like
this, we say that the lifetime `'a` **may dangle**, referring to the C
term "dangling pointer", which means a pointer to freed or invalid
memory.

However, if that same reference is stored in the field of a struct
which implements the `Drop` trait, when the struct may, during its
destructor, access the referenced value, so it's very important that
the reference be valid in that case. Put another way, if you have a
value `v` of type `Foo<'a>` that implements `Drop`, then `'a`
typically **cannot dangle** when `v` is dropped (just as `'a` would
not be allowed to dangle for any other operation).

More generally, RFC 1327 defined specific rules for which lifetimes in
a type may dangle during drop and which may not. We integrate those
rules into our liveness analysis as follows: the MIR instruction
`DROP(lvalue)` is not treated like other MIR instructions when it
comes to liveness. If it were, the value `lvalue` (and all lifetimes
found in its type) would be forced to be live. Instead, we say that
all lifetimes in `lvalue` which **cannot dangle** are live at the
point of the drop, but the other lifetimes are not considered live.

Permitting lifetimes to dangle during drop is very important! In fact,
it is essential to even the most basic non-lexical lifetime examples,
such as Problem Case #1.  After all, if we translate Problem Case #1
into MIR, we see that the reference `slice` will wind up being dropped
at the end of the block:

```rust
let mut data: Vec<i32>;
let slice: &'slice mut i32;
START {
    ...
    slice = &'borrow mut data;
    capitalize(slice);
    data.push('d');
    data.push('e');
    data.push('f');
    DROP(slice);
    DROP(data);
}
```

This poses no problem for our analysis, however, because `'slice` "may
dangle" during the drop, and hence is not considered live.

## Layer 3: Named lifetimes

APPROACH #1:

- Extend the CFG

APPROACH #2:

- Add skolemized nodes that can be included in the set
- When user writes `'a`, that is 

## Layer 4: How the borrow check works

For the most part, the focus of this RFC is on the structure of
lifetimes. But it's worth talking a bit about how to integrate
these non-lexical lifetimes into the borrow checker. In particular,
along the way, we'd like to fix two shortcomings of the borrow checker:

**First, support nested method calls like `vec.push(vec.len())`.**
Here, the plan is to continue with the `mut2` borrow solution proposed
in [RFC 2025]. This RFC does not (yet) propose one of the more
"comprehensive" solutions described in RFC 2025, such as "borrowing
for the future" or `Ref2`. The reasons why are discussed in the
Alternatives section.

- **Second, permit variables containing mutable references to be modified,
  even if their referent is borrowed.**
  
# How We Teach This
[how-we-teach-this]: #how-we-teach-this

## Terminology

In this RFC, I've opted to continue using the term "lifetime" to refer
to the portion of the program in which a reference is in active use
(or, alternatively, to the "duration of a borrow"). As the intro to
the RFC makes clear, this terminology somewhat conflicts with an
alternative usage, in which lifetime refers to the dynamic extent of a
value (what we call the "scope"). I think that -- if we were starting
over -- it might have been preferable to find an alternative term that
is more specific. However, it would be rather difficult to try and
change the term "lifetime" at this point, and hence this RFC does not
attempt do so. To avoid confusion, however, it seems best if the error
messages result from the region and borrow check avoid the term
lifetime where possible, or use qualification to make the meaning more
clear.

## Leveraging intuition: framing errors in terms of points

Part of the reason that Rust currently uses lexical scopes to
determine lifetimes is that it was thought that they would be simpler
for users to reason about. Time and experience have not borne this
hypothesis out: for many users, the fact that borrows are
"artificially" extended to the end of the block is more surprising
than not. Furthermore, most users have a pretty intuitive
understanding of control flow (which makes sense: you have to, in
order to understand what your program will do).

We therefore propose to leverage this intution when explaining borrow
and lifetime errors. To the extent possible, we will try to explain
all errors in terms of three points:

- The point where the borrow occurred (B).
- The point where the resulting reference is used (U).
- An intervening point that might have invalidated the reference (A).

We should select three points such that B can reach U and U can reach
A. In general, the approach is to describe the errors in "narrative" form:

- First, value is borrowed occurs.
- Next, the action occurs, invalidating the reference.
- Finally, the next use occcurs, after the reference has been invalidated.

This approach is similar to what we do today, but we often neglect to
mention this third point, where the next use occurs. Note that the
"point of error" remains the *second* action -- that is, the error,
conceptually, is to perform an invalidating action in between two uses
of the reference (rather than, say, to use the reference after an
invalidating action). This actually reflects the definition of
undefiend behavior more accurately (that is, performing an illegal
write is what causes undefined behavior, but the write is illegal
because of the latter use).

To see the difference, consider this erroneous program:

```
fn main() {
    let mut i = 3;
    let x = &i;
    i += 1;
    println!("{}", x);
}
```

Currently, we emit the following error:

```
error[E0506]: cannot assign to `i` because it is borrowed
 --> <anon>:4:5
   |
 3 |     let x = &i;
   |              - borrow of `i` occurs here
 4 |     i += 1;
   |     ^^^^^^ assignment to borrowed `i` occurs here
```

Here, the points B and A are highlighted, but not the point of use
U. Moreover, the "blame" is placed on the assignment. Under this RFC,
we would display the error as follows:

```
error[E0506]: cannot write to `i` while borrowed 
 --> <anon>:4:5
   |
 3 |     let x = &i;
   |              - (shared) borrow of `i` occurs here
 4 |     i += 1;
   |     ^^^^^^ write to `i` occurs here, while borrow is still active
 5 |     println!("{}", x);
   |                    - borrow is later used here
```

Another example, this time using a `match`:

```rust
fn main() {
    let mut x = Some(3);
    match &mut x {
        Some(i) => {
            x = None;
            *i += 1;
        }
        None => {
            x = Some(0); // OK
        }
    }
}
```

The error might be:

```
error[E0506]: cannot write to `x` while borrowed
 --> <anon>:4:5
   |
 3 |     match &mut x {
   |           ------ (mutable) borrow of `x` occurs here
 4 |         Some(i) => {
 5 |              x = None;
   |              ^^^^^^^^ write to `x` occurs here, while borrow is still active
 6 |              *i += 1;
   |              -- borrow is later used here
   |
```

(Note that the assignment in the `None` arm is not an error, since the
borrow is never used again.)

## Some special cases

There are some cases where the three points are not all visible
in the user syntax where we may need some careful treatment.

### Method calls

One example would be method calls:

```rust
fn main() {
    let mut x = vec![1];
    x.push(x.pop().unwrap());
}
```

We propose the following error for this sort of scenario:

```
error[E0506]: cannot write to `x` while borrowed
 --> <anon>:4:5
   |
 3 |     x.push(x.pop().unwrap());
   |     - ^^^^ ^^^^^^^^^^^^^^^^
   |     | |    write to `x` occurs here, while borrow is still in active use
   |     | borrow is later used here, during the call
   |     `x` borrowed here
```

If you are not using a method, the error would look slightly different,
but be similar in concept:

```
error[E0506]: cannot assign to `x` because it is borrowed
 --> <anon>:4:5
   |
 3 |     Vec::push(&mut x, x.pop().unwrap());
   |     ^^^^^^^^^ ------  ----------------
   |     |         |       write to `x` occurs here, while borrow is still in active use
   |     |         `x` borrowed here
   |     borrow is later used here, during the call
```

We can detect this scenario in MIR readily enough by checking when the
point of use turns out to be a "call" terminator. We'll have to tweak
the spans to get everything to look correct, but that is easy enough.

### Closures

As today, when the initial borrow is part of constructing a closure,
we wish to highlight not only the point where the closure is
constructed, but the point *within* the closure where the variable in
question is used.

## Borrowing a variable for longer than its scope

Consider this example:

```
let p;
{
    let x = 3;
    p = &x;
}
println!("{}", p);
```

In this example, the reference `p` refers to `x` with a lifetime that
exceeds the scope of `x`. In short, that portion of the stack will be
popped with `p` still in active use. In today's compiler, this is
detected during the borrow checker by a special check that computes
the "maximal scope" of the path being borrowed (`x`, here). This makes
sense in the existing system since lifetimes and scopes are expressed
in the same units (portions of the AST).  In the newer, non-lexical
formulation, this error would be detected somewhat differently. As
described earlier, we would see that a `StorageDead` instruction frees
the slot for `x` while `p` is still in use. We can thus present the
error in the same "three-point style":

```
error[E0506]: variable goes out of scope while still borrowed
 --> <anon>:4:5
   |
 3 |     p = &x;
   |          - `x` borrowed here
 4 | }
   | ^ `x` goes out of scope here, while borrow is still in active use
 5 | println!("{}", p);
   |                - borrow used here, after invalidation
```

## Errors during inference

The remaining set of lifetime-related errors come about primarily due
to the interaction with function signatures. For example:

```rust
impl Foo {
  fn foo(&self, y: &u8) -> &u8 {
    x
  }
}  
```

We already have work-in-progress on presenting these sorts of errors
in a better way (see [issue 42516][] for numerous examples and
details), all of which should be applicable here. In short, the name
of the game is to identify patterns and suggest changes to improve the
function signature to match the body (or at least diagnose the problem
more clearly).

[issue 42516]: https://github.com/rust-lang/rust/issues/42516

Whenever possible, we should leverage points in the control-flow and
try to explain errors in "narrative" form.

# Drawbacks
[drawbacks]: #drawbacks

There are very few drawbacks to this proposal. The primary one is that
the **rules** for the system become more complex. However, this
permits us to accept a large number of more programs, and so we expect
that **using Rust** will feel simpler. Moreover, experience has shown
that -- for many users -- the current scheme of tying reference
lifetimes to lexical scoping is confusing and surprising.

# Alternatives
[alternatives]: #alternatives

### Alternative formulations of NLL

During the runup to this RFC, a number of alternate schemes and
approaches to describing NLL were tried and discarded.

### Different "lifetime modes"

In the discussion about nested method calls ([RFC 2025]), there were
various proposals that were aimed at accepting code like the following:

```rust
let tmp0 = &mut vec;
let tmp1 = vec.len(); // does a shared borrow of vec
Vec::push(tmp0, tmp1);
```

This is because code like that would be the naive desugaring of
`vec.push(vec.len())`. RFC 2025 proposed instead an alternative
desugaring for `vec.push(vec.len())` which allowed us to accept such
borrows when in method-call form (but not otherwise).


# Unresolved questions
[unresolved]: #unresolved-questions

None at present.

# Endnotes

<a name="temporaries"></a>

**1.** Scopes always correspond to blocks with one exception: the
scope of a temporary value is sometimes the enclosing
statement.

[RFC 2025]: https://github.com/rust-lang/rfcs/pull/2025

