This is a tiny Rust crate that works around a common annoyance with the borrow checker.

Sometimes, you try to call a method on an object that takes `&mut self`, but you need to transiently borrow the object to compute the parameter(s) of the method. For example:

```rust
let mut v = vec![1, 2, 3];
v.reserve(v.capacity()); // double the capacity of the vector
```

This fails to compile with an error saying that you can't borrow `v` mutably (for `reserve`) while also borrowing it immutably (for `capacity`). But logically, the parameters are computed before the method call starts, so what's the issue? The immutable borrow should be over! Unfortunately, the borrow checker doesn't see it that way, since its regions are all "lexical" -- corresponding to an actual range of characters in the source code. We can fix the error by rewriting the code like this:

```rust
let mut v = vec![1, 2, 3];
let v_cap = v.capacity();
v.reserve(v_cap); // double the capacity of the vector
```

Now the immutable borrow begins and ends before the mutable one starts, and borrowck is happy. But this is tedious. If only a macro could do it for us.

```rust
#[macro_use] extern crate unborrow;

let mut v = vec![1, 2, 3];
unborrow!(v.reserve(v.capacity()));
```

By the way, this issue could conceivably disappear if "non-lexical borrowing" is ever added to the Rust compiler.

As an aside, this macro would be interesting to read if you are curious about details of macro hygiene in Rust. The macro exploits hygiene to achieve a "gensym"-like facility: to precompute all arguments to a function in bindings, we need unique names for each of those bindings that don't collide with any other variables in scope. If you look at the pretty-expanded form of the source, it looks like all the bindings are named "arg". But in fact, they are all different because they are tagged with different hygiene contexts. See the source code for more.

