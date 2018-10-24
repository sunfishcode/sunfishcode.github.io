# Canonicalization

Canonicalization and canonical forms are one dimension of organizing the
work of an optimizing compiler.

## Intro

A lot of code constructs can be written in multiple ways. For example:

```
   x + 4
   4 + x
   (x + 2) + 2
```

*Canonicalization* means picking one of these forms to
be the *canonical form*, and then going through the program and
rewriting all constructs which are equivalent to the canonical
form into the canonical form.

In the case of add, it's common to pick the form that has a constant
on the right side, so we'd rewrite all
these constructs to `x + 4`.

### Why is canonicalization useful?

The goal of canonicalization is *to make subsequent optimizations more
effective*. This is a key point, and we'll get into some of the subtleties
below. But there are a lot of cases where it's just obviously a good thing
to do. It means that subsequent optimizations that look for specific patterns of code
only have to look for the canonical forms, rather than all forms.

Another way of saying this is, having a canonicalization pass is a
way of factoring out the parts in a compiler that know all the different
forms `x + 4` could take, so that most optimization passes don't have
to worry about this. They don't have to look for `4 + x`, because they
can assume that looking for `x + 4` covers that. Handy!

### How do we choose a canonical form?

Sometimes it's easy. The canonical form for `2 + 3` is `5`, because
that's clearly simpler in every possible way.

Sometimes it's an arbitrary choice. It often doesn't matter than much
whether one picks `4 + x` over `x + 4`, but it is helpful to pick one
or the other, so sometimes it's just human aesthetics.

It's tempting to pick whatever form would be fastest, or *optimal*, on
the target machine. And indeed, sometimes what's fastest aligns with what's
simplest. `2 + 3` canonicalizing `5` is typically such a case. But
sometimes it doesn't.

### Yeah so what about `x * 2`...

`x * 2` is equivalent to `x + x`; which of these should be
the canonical form? It might seem like we might want to say: pick whatever's
optimal for the target architecture. Addition is generally
faster than multiplication, so that would suggest we pick `x + x` as
the canonical form.

But, `x + x` can actually make things harder for subsequent
optimizations, because it means that now `x` has multiple uses. Having multiple uses
makes some optimizations more complex -- in terms of the dependence graph,
this is a DAG rather than a tree, and trees are generally simpler to
work with. So maybe `x * 2` is actually a better canonical form, even if it's a worse optimal form.

That said, we might consider canonicalizing this to `x << 1`, which has only one use of `x`,
and has the nice property of making it as obvious as
possible that the least significant bit of the result is zero. That way,
we don't have to have as much random knowledge of multiplication by
powers of two strewn throughout the compiler.

Efficiency on the target machine still matters, but we can defer thinking
about that until codegen, where it's no longer as important to enable
subsequent optimizations. At that point, we're going to start caring about picking between `+`,
`*`, and `<<` based on which one executes fastest.

The basic philosophy
of canonicalization says that canonical forms should be translated into
optimal forms toward the back of the compiler, after all mid-level
optimizations which benefit from canonical form are done. It's also
worth noting that codegen itself benefits from having the
code coming into it be in canonical form, so that it doesn't have to
recognize all the ways to write `x << 1`, and can just recognize one
pattern for that and emit the optimal code for it.

This is often a source of confusion: Is canonicalization
the same as optimization? It's often done as part of the "optimizer", and
many of the things it does produce more optimal code directly. But
ultimately, in its purest form, canonicalization just focuses on
removing unnecessary variation so that subsequent optimizations can
be simpler.

> Canonical form, canonical form<br>
Canonical form hates optimal form<br>
They have a fight, canonical wins<br>
Canonical form…

(sung to the tune of "Particle Man" by They Might Be Giants)

### Sometimes it's ambiguous: redundancy elimination

Is redundancy elimination a canonicalization or an optimization?

It's certainly simpler to compute a given value once and reuse the value, rather than compute it twice.
But does that aid subsequent optimizations? It depends.

A case where it does aid subsequent optimizations is when it eliminates redundant memory accesses.
That way, it's essentially saying that no later passes have to even ask what the dependencies are
for a given memory access, because the memory access has been eliminated.

A case where it doesn't is where it can take expression trees where everything has a single use and
give some values multiple uses. Some kinds of optimization passes are harder to do on DAGs than on
trees, so this can result in pessimizations in some cases, depending on what kinds of things the
rest of the compiler is doing.

However, we typically do think of redundancy elimination as being a canonicalization.
It's trivial to convert multiple-use values into single-use values by duplicating code,
while going the other direction requires some analysis.

### Even more ambiguous: inlining

Inlining can act like canonicalization, especially in cases where the inlined function body can
be optimized away. However, thinking of inlining in terms of canonicalization doesn't lead to a natural threshold. Should we maximally
inline everything as far as possible? Or should we do the reverse and maximally outline and deduplicate
outlined functions? Neither extreme is particularly practical, so most compilers use heuristics in
practice rather than having a rigid definition of canonical form relative to calls.

That said, even though there's no clear boundary, we can imagine a rough guideline.
Think of old-school C code, written at a time when "C is a portable assembly language"
was more true than it is today, where calls that were important to inline were
written as macros. In many ways, one of the jobs of compilers for higher-level languages
is to compile them down to roughly this level, so that they can be optimized using
optimization techniques which work well at this level. In theory, we could define the
task of a canonicalizing inliner to just be to inline code down to what it would have
looked like if that same code had been written in old-school C (at least with
respect to inlining). That's not very pure, and it's difficult to precisely describe,
but it's an intuitive and relatively practical compromise between extremes.

### Canonical form isn't just for arithmetic expressions!

Everything in an IR may be subjected to canonicalization.

For example, a control flow canonicalization might involve sorting the
basic blocks of a program into Reverse Post-Order (RPO). Doing this is
a simple way to ensure that the code is optimized the same regardless
of how the user organizes the code inside their functions.

Dead code elimination is also a kind of canonicalization. The canonical
form for dead code is no code.

In aggressive loop-transforming compilers, another form of canonicalization
is to maximally fission loops into as many parts as possible, and then
assume subsequent passes will fuse them back together into optimal
loops that effectively utilize available registers and cache space.

### Canonicalization as compression

It's often the case that smaller forms are preferred over longer
forms, so canonicalization tends to make code smaller, making it a
form of compression. Also, since it effectively reduces non-essential
entropy, it can make subsequent general-purpose compression more
effective as well.

If we could conceptually perform all theoretically possible
canonicalizations on a program, we'd end up with something related to
its [Kolmogorov Complexity](https://en.wikipedia.org/wiki/Kolmogorov_complexity)
(it may not be identical, since canonicalization puts the needs of
subsequent optimizations first, rather than absolute compression).
Maximal canonicalization is frequently impossible in practice, because
of the halting problem, but also because even in cases where it's
theoretically possible, it can require impractical amounts of computation.

That said, it is pretty fun to think that for any given program, there
is a theoretical "maximally canonical form" for that program, that
all equivalent ways of writing that program could be reduced to. It
is tempting to think of this as a kind of pure essence of the program.

### Excessive canonicalization

Canonicalization discards inessential information.
However, sometimes that information can be useful to preserve.

An example arises in instruction scheduling: Say a user writes code like

```
  x = a + b;
  y = c * d;
```

Assuming there's no aliasing going on here, there's no reason why
one of these statements has to be ordered before the other. Their
order in the user's source code is inessential information.
"Sea of nodes" style compilers may canonicalize to the point where
there is no inherent ordering between these two statements.

The compiler backend ultimately has to produce machine code, which
on conventional architectures requires the compiler to pick *some*
ordering. Compilers can be pretty smart, and can take into
consideration many things, such as available execution resources
before and after these statements to know what order a 
CPU would prefer to see them in. However, as smart as they can be,
compilers can't always find the optimal answers. Optimal instruction
scheduling is NP complete, but also, it may come down to runtime
factors that ahead-of-time compilers don't have. And 
CPU hardware performance characteristics aren't always fully
documented.

#### "Do No Harm"?

Most software has no idea how it'll be mapped on to CPU pipelines.
But some does. And it's these cases where getting the mapping right is
most important.

On such software, optimizations that make actual improvements are fine, however it can be
important that compilers not make anything *worse* than if they
had translated the code naively. Aggressively canonicalizing compilers
have a risk that they will throw away information and at the end
reconstruct a form which is worse than if they had just simply translated the code as it was written.

This, along with the ambiguous cases above, suggests that canonicalization
be used in practical rather than rigid ways.

---

## The theoretical shape of optimization.

A compiler typically starts with human-written source code.

Typical human-written code
follows various human-oriented sensibilities. Different humans may have different aesthetic sensibilities,
and this can lead to writing the same code in different ways. But of
course this isn't interesting to optimizers.

So the first then we typically do when the code hits the optimizers is
to start canonicalizing. Throw away useless fluff that humans
imbue their code with.

Canonicalizing optimizations are cascading; often doing one canonicalization
will enable more. Folding an expression to a constant may allow other
expressions to be folded to constants. Replacing a load by forwarding a value
through a prior store may create a direct edge between two expression trees
and introduce opportunities to simplify further.

So ignoring the practical concerns we mentioned above, we can imagine a
theoretical compiler that does all the canonicalization it knows how to do
up front. And while we don't realistically approach Kolmogorov Complexity
levels, we do lift the program closer to what we might think of as its
essence, the simplest form that does what it needs to do.

And then, the compiler can begin to optimize, rewriting canonical forms into
optimal forms, as it lowers the code all the way down to assembly code.

For practical reasons, it isn't always possible to build compilers in terms
of a purely canonicalizing phase and a purely optimizing phase, however this
can be a useful reference point for understanding compilers in practice.
