# First-Class I/O

@withoutBoats' makes an interesting observation about Rust in ["Notes on a smaller Rust"]:

> pure functional programming is an ingenious trick to show you can code
> without mutation, but Rust is an even cleverer trick to show you can just
> have mutation.

A particular aspect I'd like to explore here is: Can we apply this
observation to I/O?

Haskell also has an ingenious trick to do first-class I/O without mutation. So,
can we have first-class I/O in Rust that just uses mutation?

["Notes on a smaller Rust"]: https://without.boats/blog/notes-on-a-smaller-rust/

## What is First-Class I/O?

As in Haskell, first-class I/O in Rust would mean functions that do I/O would
do so through values which can be passed around the program as arguments or
return values:

```rust
fn do_some_io(f: &File) -> io::Result<()> {
    ...
}
```

`File` here is an example of a first-class value which represents a resource
that supports I/O. But unlike Haskell, instead of using monads, `File` here just
has side-effecting operations like `read` and `write`.

Of course, Rust already has lots of these kinds of types, including in the
standard library with types like `File` and `TcpStream`. And a lot of Rust
code already follows this pattern. Strictly speaking, many of the operations
don't take `&mut` references, but these types conceptually use
[interior mutability], which isn't unique to I/O.

The remaining piece that Haskell has that Rust doesn't here is that in Haskell,
*all* I/O is done through values which are passed around through the program.
Any function which does I/O says so in its signature. In Rust, when a function
has a `File` argument in its signature, you know it's going to do I/O using that
`File`, but a function which doesn't have `File` or any other I/O type might
still access files.

[interior mutability]: https://doc.rust-lang.org/reference/interior-mutability.html

## Example: Stdout

An example in Rust of code that uses global I/O is `std::io::stdout`. Any
code can call this and obtain a `Stdout` value that can do I/O without having
any mention of it in the surrounding function's signature. `Stdout` uses a
builtin mutex, even though many use cases don't need that, and is a common
performance pitfall. Diligent users can of course use `StdoutLock` to reduce
the performance impact in some cases, but what if we could eliminate the
builtin mutex altogether?

What if stdout was a value that a program would acquire once, and then pass
around to all functions that want to print to it? Using Rust's usual ownership
and borrowing rules, it wouldn't need a mutex for many use cases. And users
could of course still explicitly wrap it in a `Mutex` or similar in cases where
they really want shared access to it, just like anything else in Rust.

The [`io-streams` crate] has an implementation of this. The
[`StreamWriter::stdout`] function returns an output stream which writes to
the process' standard output. It implements the standard [`Write`] trait
so it's easy to use, and it conceptually owns its resource, so it doesn't
need a builtin mutex.

Behind the scenes, this function actually acquires a `StdoutLock` to prevent
accidental mixing of `std::io::stdout` usage with `StreamWriter::stdout`
usage, to uphold its exclusive ownership assumption.

[`Write`]: https://doc.rust-lang.org/std/io/trait.Write.html
[`StreamWriter::stdout`]: https://docs.rs/io-streams/latest/io_streams/struct.StreamWriter.html#method.stdout
[`io-streams` crate]: https://crates.io/crates/io-streams

## Example: Files

As another example, any code can do `File::open(...)` and pass it a string,
to open any file in the process' filesystem namespace, without declaring it
in a function signature.

This means that if you want to run piece of code that does this in a different
directory, the only way to do so is to create a new process, with a new
filesystem namespace. This is a very heavy-weight operation, both in terms of
performance and memory usage, but also in terms of portability and complexity.

Some codebases have a convention of having a "root" path that is passed in
that everything is relative to, which helps, but doesn't enforce that all
paths are relative to the root, or that paths don't lead outside the root
using `..`.

The [`cap-std` crate] has a [`Dir`] type, which represents a directory,
which can make filesystem access a first-class part of a function's signature,
and allow callers to specify a different directory for the I/O to happen in.
And, it also performs sandboxing, ensuring that paths relative to the `Dir`
stay within the `Dir`.

Filesystems are effectively pools of aliased mutable state, and with relatively
weak synchronization primitives. While `cap-std` doesn't address all the
problems that can arise from this, it can be one tool for helping manage
this state with standard Rust idioms.

[`cap-std` crate]: https://crates.io/crates/cap-std
[`Dir`]: https://docs.rs/cap-std/latest/cap_std/fs/struct.Dir.html

## Tangent: Rethinking "pure" functions

Rust doesn't have a way to declare functions as "pure", having no side effects.

```rust
fn this_is_pure(x: i32, y: i32) -> i32 {
   x + y
}
```

(As an aside, this function could panic on overflow; in what follows, I assume
"pure" permits panics.)

Features to enable this have been [proposed] a few times, but they haven't been
added to the language, in part because the need for such features tends to be
lower in Rust than other languages. From an optimizer perspective, this
property can often be inferred, at least in simple cases.

And from a programmer perspective, programmers don't need "pure" to know whether
arguments are mutated or not, because in Rust, these things are already
declared, reliably, in the signature. The presence or absence of `&mut` or
types with interior mutability tells you everything you need to know about
which arguments could be mutated. As such, much of what "pure" would mean would
be redundant with information which is already there:

```rust
/// Can you guess which state this function mutates?
fn increment(x: &mut i32) {
   // You're right!
   *x += 1;
}
```

The things that aren't covered by the signature are global mutable state and
I/O. Global mutable state tends to be less important in Rust than other
languages because Rust pretty strongly discourages global mutable state.

However, Rust doesn't outright prohibit global mutable state, and doesn't
really discourage global I/O, which is present even in the standard library.

So instead of a "pure", that prohibits all mutations, the more interesting
property would be what I'll call "explicit", which would mean "no global
mutable state or I/O". When used on a function with no `&mut` or
interior-mutable types in the signature, "explicit" would be the same as
"pure", and could be declared as "pure" to optimizers. However, "explicit"
could be used in functions that do have mutable arguments too, where it
would indicate that those are the only things that are mutated.

```rust
/// All mutations and I/O are accounted for!
fn this_is_explicit(f: &File, x: &mut i32) -> io::Result<()> {
    writeln!(f, "hello world")?;
    *x += 1;
    Ok(())
}
```

The "explicit" property would also serve to indicate a lack of
*ambient authority*, from a capability-oriented security perspective.

Would it make sense to add an "explicit" keyword to Rust then? Probably not
as such; it's likely that the majority of functions in Rust would qualify as
"explicit", so adding it as a function attribute would add a lot of clutter.
A more ergonomic approach might be to give crates a way to declare that all
their functions are explicit, with a way to opt out for individual functions.

A possible direction for future exploration would be to use the
[rustc driver API] to create a custom static analysis tool that could recognize
some form of syntax for this and then check that "explicit" functions don't
accidentally call "non-explicit" functions without explicit overrides, much
like `unsafe`.

[rustc driver API]: https://rustc-dev-guide.rust-lang.org/rustc-driver.html
[proposed]: https://github.com/rust-lang/rfcs/issues/1631

## Conclusion

First-class I/O points to a way of thinking about software where "the machine"
it's running on or "the process" it's running in aren't the focus. Instead of
ad-hoc conventions for coordinating access to a shared filesystem namespace or
other process-associated resources, code using first-class I/O can pass values
around to manage its I/O resources similar to how it already manages other
program resources.

First-class I/O can be a useful concept to apply broadly, such as how
[capability-oriented security] helps enable [shared-nothing linking]. It can
also be useful to apply incrementally, such as how the [`io-streams` crate]
or the [`cap-std` crate] can help parts of a program cooperate with each other
efficiently and idiomatically.

*Thanks to [Pat Hickey](https://github.com/pchickey) and
[Luke Wagner](https://github.com/lukewagner) for feedback on this post!*

[shared-nothing linking]: https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md#creating-maximally-reusable-modules
[capability-oriented security]: https://en.wikipedia.org/wiki/Capability-based_security
