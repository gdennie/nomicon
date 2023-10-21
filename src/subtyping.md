# Subtyping and Variance

Rust uses lifetimes to track the relationships between references and targets.
However, a naive implementation of lifetimes would be either too restrictive,
or permit undefined behavior.

In order to allow flexible usage of lifetimes
while also preventing their misuse, Rust uses **subtyping** and **variance**
to merge different lifetimes into a single lifetime when required.

Let's start with an example of a function requiring one lifetime over 
two arguments but invoked with two distinct lifetimes.

```rust
// Note: debug expects two parameters with the *same* lifetime
fn debug<'a>(a: &'a str, b: &'a str) {
    println!("a = {a:?} b = {b:?}");
}

fn main() {
    let hello: &'static str = "hello";
    {
        let world = String::from("world");
        let world = &world; // 'world has a shorter lifetime than 'static
        debug(hello, world);
    }
}
```

In a conservative implementation of lifetimes, since `hello` and `world` have different lifetimes,
we might see the following error:

```text
error[E0308]: mismatched types
 --> src/main.rs:10:16
   |
10 |         debug(hello, world);
   |                      ^
   |                      |
   |                      expected `&'static str`, found struct `&'world str`
```

This would be rather unfortunate. In this case,
what we want is to accept any type that lives *at least as long* as `'world`.
Let's try using subtyping with our lifetimes.

## Subtyping

Subtyping is the idea that one type, the sub type, can be used in place of another.

Let's define that the type `Sub` is a subtype of `Super` (we'll be using the notation `Sub : Super` throughout this chapter).

What this suggests is that the set of *requirements* that `Super` supports
are completely satisfied by `Sub`. `Sub` may then support more requirements.

Now, in order to use subtyping with lifetimes, we need to define the requirement of a lifetime. 
It is one simple thing...

> `'a` delimits a path of execution often simplified as a region of code.

Now that we have a defined set of requirements for lifetimes, we can define how they relate to each other:

> `'long : 'short` if and only if `'long` defines a region of code that **completely contains** `'short`.

`'long` may define a region larger than `'short`, but that still fits our definition.

> As we will see throughout the rest of this chapter,
subtyping is a lot more complicated and subtle than this,
but this simple rule is a very good 99% intuition.
And unless you write unsafe code, the compiler will automatically handle all the corner cases for you.

> But this is the Rustonomicon. We're writing unsafe code,
so we need to understand how this stuff really works, and how things can get messed up.

Going back to our example above, we can say that `'static : 'world`.
For now, let's also accept the idea that subtypes of lifetimes can be passed through references
(more on this in [Variance](#variance)),
_e.g._ `&'static str` is a subtype of `&'world str`, then we can "downgrade" `&'static str` into a `&'world str`.
With that explanation we see the example above does compile:

```rust
fn debug<'a>(a: &'a str, b: &'a str) {
    println!("a = {a:?} b = {b:?}");
}

fn main() {
    let hello: &'static str = "hello";
    {
        let world = String::from("world");
        let world = &world; // 'world has a shorter lifetime than 'static
        debug(hello, world); // hello silently downgrades from `&'static str` into `&'world str`
    }
}
```

## Variance

We glossed over the fact that `'static < 'b` implied that `&'static T < &'b T`. The implicit extension of the concept
is a property known as _variance_.
It's not always as simple as this example though. To understand that, let's try to extend this example a bit:

We call `assign` to set the `hello` reference to _point_ to `world`.
Then `world` goes out of scope at the end of the inner block in which
it is defined and before the `hello` in the println!

```rust,compile_fail,E0597
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    {
        let world = String::from("world");
        assign(&mut hello, &world);
    }
    println!("{hello}"); // use after free 😿
}
```

This is a classic use-after-free bug!

Our first instinct might be to blame the `assign` impl, but there's really nothing wrong here.
It shouldn't be surprising that we might want to assign a `T` into a `T`.

The problem is that we cannot assume that `&mut &'static str` and `&mut &'b str` are compatible.
This means that `&mut &'static str` **cannot** be a *subtype* of `&mut &'b str`,
even if `'static` is a subtype of `'b`.

Variance is the concept that Rust borrows to define relationships about subtypes 
through their generic parameters.

> NOTE: For convenience we will define a generic type `F<T>` so
> that we can easily talk about the relationship between
> the various instance types of `T`.
> Hopefully this is clear in context.

`F`'s *variance* describes how subtypes relationship between its inputs (generic arguments) affects
subtyping between instances of its output (constructed type). 
There are three kinds of variance in Rust. Given `Super` and `Sub`, a subtype of `Super`, the following holds:

* `F` is **covariant** when `F<Sub>` is a subtype of `F<Super>` (the subtype property is passed through)
* `F` is **contravariant** when `F<Super>` is a subtype of `F<Sub>` (the subtype property is "inverted")
* `F` is **invariant** otherwise (no subtyping relationship exists)

If we remember from the above examples,
it was ok for us to treat `&'a T` as a subtype of `&'b T` if `'a : 'b`;
`&'a T` will be around as long as `&'b T` and so we can 
assign `&'a T` to `&'b T` as the immutable reference is tolerant of 
the shorter lifetime constraint.
Therefore we can say that `&'a T` is *covariant* over `'a`.

However, we saw that it was _not ok_ to similarly presume
`&mut &'a T` is a subtype of `&mut &'b T` simply because `'a : 'b`
and have `&mut &'a T` assignable to `&mut &'b T`. Similarly,
`&mut &'b T` is not assignable to `&mut &'a T` because it
is not gauranteed to live long enough. Consequently, the 
lifetimes have to be the same.
Therefore, `&mut T` is *invariant* over `T`

Here is a table of some other generic types and their variances:

|                 |     'a    |         T         |     U     |
|-----------------|:---------:|:-----------------:|:---------:|
| `&'a T `        | covariant | covariant         |           |
| `&'a mut T`     | covariant | invariant         |           |
| `Box<T>`        |           | covariant         |           |
| `Vec<T>`        |           | covariant         |           |
| `UnsafeCell<T>` |           | invariant         |           |
| `Cell<T>`       |           | invariant         |           |
| `fn(T) -> U`    |           | **contra**variant | covariant |
| `*const T`      |           | covariant         |           |
| `*mut T`        |           | invariant         |           |

Some of these can be explained simply in relation to the others:

* `Vec<T>` and all other owning pointers and collections follow the same logic as `Box<T>`
* `Cell<T>` and all other interior mutability types follow the same logic as `UnsafeCell<T>`
* `UnsafeCell<T>` having interior mutability gives it the same variance properties as `&mut T`
* `fn(T) -> U` (see below)
* `*const T` follows the logic of `&T`
* `*mut T` follows the logic of `&mut T` (or `UnsafeCell<T>`)

> NOTE: the *only* source of contravariance in the language is in the arguments to
> a function where, if `B: A` then `F(A) : F(B)`.
> For instance, if `B` is an `i32` and `A` is an `i64` then any value of `B` is assignable to `A`.
> However, an `F(i32)` cannot stand in for `F(i64)` as it would not be able to
> handle receiving expected `i64` values outside the range of `i32`.
> On the other hand, an `F(i64)` can indeed stand in for an `F(i32)` since it is fully capable of
> handling the entire `i32` expected input input.
> Similarly for lifetimes, if `'b: 'a` then `F(&'a T): F(&'b T)` 
> because a function that requires access for a shorter time can
> stand in for a function that requires access for a longer time.

For more types, see the ["Variance" section][variance-table] on the reference.

[variance-table]: ../reference/subtyping.html#variance

Now that we have some more formal understanding of variance,
let's go through some more examples in more detail.

```rust,compile_fail,E0597
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    {
        let world = String::from("world");
        assign(&mut hello, &world);
    }
    println!("{hello}");
}
```

And what do we get when we run this?

```text
error[E0597]: `world` does not live long enough
  --> src/main.rs:9:28
   |
6  |     let mut hello: &'static str = "hello";
   |                    ------------ type annotation requires that `world` is borrowed for `'static`
...
9  |         assign(&mut hello, &world);
   |                            ^^^^^^ borrowed value does not live long enough
10 |     }
   |     - `world` dropped here while still borrowed
```

Good, it doesn't compile! Let's break down what's happening here in detail.

First let's look at the `assign` function:

```rust
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}
```

All it does is take a mutable reference and a value and overwrite the target with it.
What's important about this function is that it creates a type equality constraint. It
clearly says in its signature the target and the value must be the *exact same* type.

Meanwhile, in the caller we pass in `&mut &'static str` and `&'world str`
where `T` is `&'static str` and `&'world str`, respectively.
However, because `&mut T` is invariant over `T`, the compiler concludes it can't apply 
any the available covariance subtyping relationship of `'static : 'world`
to the first argument, and so `T` must remain interpreted as exactly `&'static str`.
The result is a compiler error.

The above is counter to the `&T` case:

```rust
fn debug<T: std::fmt::Debug>(a: T, b: T) {
    println!("a = {a:?} b = {b:?}");
}
```

where similarly `a` and `b` must have the same type `T`.
But since `&'a T` *is* covariant over `'a`, we are allowed to perform subtyping.
So the compiler decides that `&'static str` can become `&'b str` if and only if
`&'static str` is a subtype of `&'b str`, which will hold if `'static <: 'b`.
This is true, so the compiler is happy to continue compiling this code.

> invariance exists when covariance and contravariance is occuring in the same type expression.
> In the case of `&'a mut T`, `'a` is covariant, which is to say because it is bigger (lives longer)
> it can be assigned to its supertype. On the other hand, `mut T`, is contravariant and states
> `T` is bigger (`i64` to `i32`), it _cannot_ be assigned to its supertype; however, its supertype
> can be assigned to it. As a result, this combination of variance in `&'a mut T` forces invariance.

Consequently, the argument for why it's ok for Box (and Vec, HashMap, etc.) to be covariant is pretty 
similar to the argument for why it's ok for lifetimes to be covariant: as soon as you try to stuff 
them in something like a mutable reference, they inherit invariance and you're 
prevented from doing anything bad.

However Box makes it easier to focus on the by-value aspect of references that we partially glossed over.

Unlike a lot of languages which allow values to be freely aliased at all times, Rust has a very strict rule: if you're allowed to mutate or move a value, you are guaranteed to be the only one with access to it.

Consider the following code:

```rust,ignore
let hello: Box<&'static str> = Box::new("hello");

let mut world: Box<&'b str>;
world = hello;
```

There is no problem at all that we will forget that `hello` is `'static`
when it is assigned to `world: Box<&'b str>` at `world = hello`
because as soon as `hello` is moved
**we will have destroyed `hello`, the only other thing in the universe that was connected and 
could affect or dependent on the value in any way**! 

Only one thing left to explain: function pointers.

To see why `fn(T) -> U` should be covariant over `U`, consider the following signature:

<!-- ignore: simplified code -->
```rust,ignore
fn get_str() -> &'a str;
```

This function claims to produce a `str` bound by some liftime `'a`. As such, it is perfectly valid to
provide a function with the following signature instead:

<!-- ignore: simplified code -->
```rust,ignore
fn get_static() -> &'static str;
```

So when the function is called, all the caller is expecting is a `&str` 
which lives at least the lifetime of `'a`.
It doesn't matter if it can actually live longer.

However, the same logic does not apply to *arguments*. Consider trying to satisfy:

<!-- ignore: simplified code -->
```rust,ignore
fn store_ref(&'a str);
```

with:

<!-- ignore: simplified code -->
```rust,ignore
fn store_static(&'static str);
```

The first function can accept any string reference as long as it lives at least for `'a`,
but the second cannot accept a string reference that lives for any duration less than `'static`.
This is a conflict.
Covariance doesn't work here. But if we flip it around, contravariance actually *does*
work! If we need a function that can expects `&'static str`, a function that can handle *any* reference lifetime
will surely work fine.

Let's see this in practice

```rust,compile_fail
# use std::cell::RefCell;
thread_local! {
    pub static StaticVecs: RefCell<Vec<&'static str>> = RefCell::new(Vec::new());
}

/// saves the input given into a thread local `Vec<&'static str>`
fn store(input: &'static str) {
    StaticVecs.with(|v| {
        v.borrow_mut().push(input);
    })
}

/// Calls the function with it's input (must have the same lifetime!)
fn demo<'a>(input: &'a str, f: fn(&'a str)) {
    f(input);
}

fn main() {
    demo("hello", store); // "hello" is 'static. Can call `store` fine

    {
        let smuggle = String::from("smuggle");

        // `&smuggle` is not static. If we were to call `store` with `&smuggle`,
        // we would have pushed an invalid lifetime into the `StaticVecs`.
        // Therefore, `fn(&'static str)` cannot be a subtype of `fn(&'a str)`
        demo(&smuggle, store);
    }

    StaticVecs.with(|v| {
        println!("{:?}", v.borrow()); // use after free 😿
    });
}
```

And that's why function types, unlike anything else in the language, are
**contra**variant over their arguments.

Now, this is all well and good for the types the standard library provides, but
how is variance determined for types that *you* define? A struct, informally
speaking, inherits the variance of its fields. If a struct `MyType`
has a generic argument `A` that is used in a field `a`, then MyType's variance
over `A` is exactly `a`'s variance over `A`.

However if `A` is used in multiple fields:

* If all uses of `A` are covariant, then MyType is covariant over `A`
* If all uses of `A` are contravariant, then MyType is contravariant over `A`
* Otherwise, MyType is invariant over `A`

```rust
use std::cell::Cell;

struct MyType<'a, 'b, A: 'a, B: 'b, C, D, E, F, G, H, In, Out, Mixed> {
    a: &'a A,     // covariant over 'a and A
    b: &'b mut B, // covariant over 'b and invariant over B

    c: *const C,  // covariant over C
    d: *mut D,    // invariant over D

    e: E,         // covariant over E
    f: Vec<F>,    // covariant over F
    g: Cell<G>,   // invariant over G

    h1: H,        // would also be covariant over H except...
    h2: Cell<H>,  // invariant over H, because invariance wins all conflicts

    i: fn(In) -> Out,       // contravariant over In, covariant over Out

    k1: fn(Mixed) -> usize, // would be contravariant over Mixed except..
    k2: Mixed,              // invariant over Mixed, because invariance wins all conflicts
}
```
