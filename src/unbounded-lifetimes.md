# Unbounded Lifetimes

Unsafe code can often end up producing references or lifetimes out of thin air.
Such lifetimes come into the world as *unbounded*. The most common source of
this is taking a reference to the target of a raw pointer. This necessarily produces a
reference with an unbounded lifetime since the compiler cannot discern the lifespan of the target.
Such a lifetime becomes as big as context demands. 
This is in fact more powerful than simply becoming `'static`, because
for instance `&'static &'a T` will fail to typecheck because we are trying
to obtain a `&'static` lifetime reference to a less persistent target. 
However, an unbound lifetime
will perfectly mold into `&'a &'a T` as needed. Nonetheless, For most intents and
purposes, such an unbounded lifetime can be regarded as `'static`.

Still, almost no runtime constructed references are `'static`, so this is probably wrong.

`transmute` and `transmute_copy` are the two other primary sources of unbound lifetimes. 
One should endeavor to
bound an unbounded lifetime as quickly as possible, especially across function
boundaries.

The easiest way to avoid unbounded lifetimes is to use lifetime elision at the
function boundary. If an output lifetime is elided, then it *will have been* bounded by
an input lifetime somehow. Of course it might be bounded by the *wrong* input lifetime, but
this will usually just cause a compiler error, rather than allow memory safety
to be trivially violated.

Given a function, any output lifetimes that don't derive from inputs are
unbounded; and so, here the function stipulates a lifetime with which the
result is bound. For instance:

<!-- no_run: This example exhibits undefined behavior. -->
```rust,no_run
fn get_str<'a>(s: *const String) -> &'a str {
    unsafe { &*s }
}

fn main() {
    let soon_dropped = String::from("hello");
    let dangling = get_str(&soon_dropped);
    drop(soon_dropped);
    println!("Invalid str: {}", dangling); // Invalid str: gӚ_`
}
```

Within a function, bounding lifetimes is more error-prone. The safest and easiest
way to bound a lifetime is to return it from a function with a bound lifetime.
However if this is unacceptable, the reference can be placed in a location with
a specific lifetime. Unfortunately it's impossible to name all lifetimes involved
in a function.
