# Lifetimes

Rust enforces these rules through a compile time (type system) feature called *lifetimes*. 
Lifetimes are named regions of code that a reference is valid for. Those regions
may be fairly complex, as they correspond to paths of execution
in the program. There may even be holes in these paths of execution,
as it's possible to invalidate a reference as long as it's reinitialized
before it's used again. Types which contain references (or pretend to, see PhantomData)
may also be tagged with lifetimes so that the Rust compiler can prevent them from
being invalidated as well.

In most of our examples, lifetimes will coincide with scope. This is
because our examples are simple. The more complex cases where they don't
coincide are described below.

Within a function body, Rust generally doesn't let you explicitly name the
lifetimes involved. This is because it's generally not really necessary
to talk about lifetimes in a local context; Rust has all the information and
can work out everything as optimally as possible.

However, once you cross the function boundary and thus have a relationship with an 
external scope, you need to start talking about lifetimes. 
Lifetimes are denoted with an apostrophe: `'a`, `'static`. 

To dip our toes with lifetimes, we're going to pretend that we're actually allowed
to label scopes with lifetimes, and desugar the examples used at the start of
this chapter.

Originally, our examples made use of *aggressive* sugar -- high fructose corn
syrup even -- around scopes and lifetimes, because writing everything out
explicitly is *extremely noisy*. All Rust code relies on aggressive inference
and elision of "obvious" things.

One particularly interesting piece of sugar is that each `let` statement
implicitly introduces a scope. For the most part, this doesn't really matter.
However, it does matter for variables of types that reference another. 
As a simple example, let's completely desugar this simple piece of Rust code:

```rust
let x = 0;
let y = &x;
let z = &y;
```

The borrow checker always tries to minimize the extent of a lifetime, so it will
likely desugar to the following:

<!-- ignore: desugared code -->
```rust,ignore
// NOTE: `'a: {` is not valid syntax but merely labels the lifetime scope of the block
// NOTE: `&'b x` is not valid syntax but expresses the lifetime associated with the variable, `x`
'a: {
    let x: i32 = 0;
    'b: {
        // lifetime used is 'b because that's good enough.
        let y: &'b i32 = &'b x;
        'c: {
            // ditto on 'c
            let z: &'c &'b i32 = &'c y; // "a reference to a reference to an i32" (with lifetimes annotated)
        }
    }
}
```

Wow. That's... awful. Let's all take a moment to thank Rust for making this easier.

While the borrow checker always tries to use the smallest lifetime, passing references 
from inner scope to outer scope will cause Rust to infer the larger lifetime, when possible.

```rust
let x = 0;
let z;
let y = &x;
z = y;
```

<!-- ignore: desugared code -->
```rust,ignore
'a: {
    let x: i32 = 0;
    'b: {
        let z: &'b i32;
        'c: {
            // Must use 'b here because the reference to x that is in scope 'a is
            // being passed to the scope 'b.
            let y: &'b i32 = &'b x;
            z = y;
        }
    }
}
```

## Example: references that outlive referents

Alright, let's look at some of those examples from before:

```rust,compile_fail
fn as_str(data: &u32) -> &str {
    let s = format!("{}", data);
    &s
}
```

desugars to:

<!-- ignore: desugared code -->
```rust,ignore
fn as_str<'a>(data: &'a u32) -> &'a str {
    'b: {
        let s = format!("{}", data);
        return &'a s;
    }
}
```

The signature of `as_str` takes a reference to a `u32` with *some* lifetime, and
promises that it can produce a reference to a `str` that is valid for just as long.
Already we can see why this signature might be trouble since it basically implies
that we're going to be providing a reference to a `str` somewhere in the scope 
of the caller or earlier. That's a bit of a tall order.

We then proceed to compute the `String`, `s`, and return a reference to it. Since
the contract of our function says the reference must outlive `'a`, that's the
lifetime we must infer for the reference. Unfortunately, `s` was defined within the
scope `'b` so the only way a reference to it is sound is if `'b` contains `'a` -- which is
clearly not the case since `'a` must contain the function call itself. We have therefore
created a reference whose lifetime outlives its target, which is *literally*
the first thing we said that references cannot do. Consequently, the compiler rightfully blows
up in our face.

To make this more clear, we can expand the example:

<!-- ignore: desugared code -->
```rust,ignore
fn as_str<'a>(data: &'a u32) -> &'a str {
    'b: {
        let s = format!("{}", data);
        return &'a s
    }
}

fn main() {
    'c: {
        let x: u32 = 0;
        'd: {
            // An anonymous scope is introduced because the borrow does not
            // need to last for the whole scope x is valid for. The return
            // of as_str must find a str somewhere before this function
            // call. Obviously not happening.
            println!("{}", as_str::<'d>(&'d x));
        }
    }
}
```

Shoot!

Of course, the right way to write this function is as follows:

```rust
fn to_string(data: &u32) -> String {
    format!("{}", data)
}
```

We must produce an owned value inside the function to return it! The only way
we could have returned an `&'a str` would have been if it was an alias to the
`&'a u32` memory location. However, a `u32` is not also the `utf8` representation
of its value.


Actually we could have also just returned a string literal, which, as a global or static,
can be considered to reside at the bottom of the stack; though this limits
our implementation to a predetermine fixed set of possible values.

```rust
fn to_string(data: &u32) -> &str {
    match data {
        0 -> "0",
        1 -> "1",
        ...
    }
}
```

## Example: aliasing a mutable reference

How about this other example:

```rust,compile_fail
let mut data = vec![1, 2, 3];
let x = &data[0];
data.push(4);
println!("{}", x);
```

<!-- ignore: desugared code -->
```rust,ignore
'a: {
    let mut data: Vec<i32> = vec![1, 2, 3];
    'b: {
        // 'b is as big as we need this borrow to be
        // (just need to get to `println!`)
        let x: &'b i32 = Index::index::<'b>(&'b data, 0);
        'c: {
            // Temporary scope because we don't need the
            // &mut to last any longer.
            Vec::push(&'c mut data, 4);
        }
        println!("{}", x);
    }
}
```

The problem here is a bit more subtle and interesting. We want Rust to
reject this program for the following reason: We have a live shared reference `x`
to a descendant of `data` while we are trying to take a mutable reference to `data`
in `push`. However, `push` requires a mutable reference and mutable references
must be exclusive. As a result, the *second* rule of references would be violated.

However this is *not at all* how Rust reasons that this program is bad. Rust
doesn't understand that `x` is a reference to a subpath of `data`. It doesn't
understand `Vec` at all. What it *does* see is that `x` has to live for `'b` in
order to become printed by `println!`. As such, the signature of `Index::index` 
must produce a reference to `data` that survives the scope of `'b`. Next, it see us try 
to call `push` in the scope of `'c` and thereby try to make a `&'c mut data`. 
But Rust knows that `'c` is contained within `'b` and so rejects our program 
because the `&'b data` must still be alive in which case `&'c mut data`
cannot be exclusive!

Here we see that the lifetime system is much more coarse than the reference
semantics we're actually interested in preserving. For the most part, *that's
totally ok*, because it keeps us from spending all day explaining our program
to the compiler. However it does mean that several programs that are totally
correct with respect to Rust's *true* semantics are rejected because lifetimes
are too dumb.

## The area covered by a lifetime

A reference (sometimes called a *borrow*) is *alive* from the place it is
created to its last use. The borrowed value, the target, only needs to outlive 
live borrows. This looks simple, but there are a few subtleties.

The following snippet compiles, because after printing `x`, it is no longer
needed, so it doesn't matter if it is dangling or aliased (even though the
variable `x` *technically* exists to the very end of the scope).

```rust
let mut data = vec![1, 2, 3];
let x = &data[0];
println!("{}", x);
// This is OK, x is no longer needed
data.push(4);
```

However, if the value has a destructor, the destructor is run at the end of the
scope. And running the destructor is considered a use ‒ obviously the last one.
So, this will *not* compile.

```rust,compile_fail
#[derive(Debug)]
struct X<'a>(&'a i32);

impl Drop for X<'_> {
    fn drop(&mut self) {}
}

let mut data = vec![1, 2, 3];
let x = X(&data[0]);
println!("{:?}", x);
data.push(4);
// Here, the destructor is run and therefore this'll fail to compile.
```

One way to convince the compiler that `x` is no longer valid is by using `drop(x)` before `data.push(4)`.

Furthermore, there might be multiple possible last uses of the borrow, for
example in each branch of a condition.

```rust
# fn some_condition() -> bool { true }
let mut data = vec![1, 2, 3];
let x = &data[0];

if some_condition() {
    println!("{}", x); // This is the last use of `x` in this branch
    data.push(4);      // So we can push here
} else {
    // There's no use of `x` in here, so effectively the last use is the
    // creation of x at the top of the example.
    data.push(5);
}
```

A lifetime can have a pause in it. Or you might look at it as two distinct
borrows tied to the same variable. This often happens with
loops in which some outer scope variable is reasigned within the loop
to become used in the header of the loop within the outer scope each
time the loop restarts its block.

```rust
let mut data = vec![1, 2, 3];
// This mut allows us to change where the reference points to
let mut x = &data[0];

println!("{}", x); // Last use of this borrow
data.push(4);
x = &data[3]; // We start a new borrow here
println!("{}", x);
```

Historically, Rust kept the borrow alive until the end of scope, so these
examples might fail to compile with older compilers. Also, there are still some
corner cases where Rust fails to properly shorten the life of the borrow
and fails to compile even when it looks like it should. These issues
will be solved over time.
