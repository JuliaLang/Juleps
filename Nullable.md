# JULEP Nullable

- **Title:** Nullable Redesign
- **Authors:** John Myles White <<jmw@johnmyleswhite.com>>
- **Created:** December 9, 2016
- **Status:** work in progress

## Abstract

The representation of null-like constructs in Julia is too complicated. We
currently have all of the following kinds of null-like constructs:

* A ` Void` type with a singleton value `nothing`,
* A `Nullable` type,
* An `#undef` quasi-value that reflects the use of null pointers in Julia's
    implementation.

This Julep describes a proposal to coalesce all of these concepts into one.

The proposal is to replace everything with a new version of the `Nullable`
type. This new version of `Nullable` will involve both changing the
implementation of our `Nullable` type and clarifying the semantics of this
type by laying out a thorough description of how the type would be used in
practice.

The core goals for this Julep are to:

* Simplify the language by replacing many similar constructs with a single
    construct.
* Ensure that all representations of `null` are first-class citizens of the
    language. In particular, we will resolve the problems with `#undef`, which
    is a quasi-value that can be present in types, but cannot be inserted into
    them.
* Resolve the lifting problem for `Nullable` types by formalizing
    general-purpose, extensible machinery for lifting.
* Make it possible to write optimized code for `Nullable` types that does not
    call into type inference via `return_types` or require the use of
    `promote_type`.

The optional goals for this Julep are to:

* Replace the `NullableArray` type with `Array{Nullable}`, while maintaining
    the ability to pass the raw values storage unit via FFI in the way that
    `NullableArray` currently allows.

The prerequisites for this Julep are the following:

* Optimizations to the Julia compiler to handle `Union{S, T}` as efficiently as
    our current `Nullable` type handles the union of `S` and `Void`. These
    compiler changes may include both memory layout changes and optimization
    of run-time dispatch among K pre-specified types into simple branches.

## Step 1: Redefining the Nullable Type

Assuming that compiler optimizations are put in place for union types, the
first step for this Julep is to replace the concrete type `Nullable` with an
optimized union type. The options for replacing `Nullable{T}` are:

* `Union{T, Void}`
* `Union{Some{T}, Void}`
* `Union{T, Null{T}}`

The `Union` type would have the same semantics as unions currently have. Their
implementation in the scalar case would likely match that of a possible
`TaggedUnion` type, which represents a discriminated union, but is a concrete
type that is not a parent of `T`, `Void`, etc.

Deciding between these options requires balancing a variety of concerns. As
we currently understand them, the arguments are:

### `Union{T, Void}` Pros & Cons

When a function returns `Union{T, Void}`, the user can immediately apply all
methods defined on `T` without any unwrapping. This is very convenient and
does not require the introduction of specialized methods.

The counter-argument is that such code will only work if the methods being
called are defined for both `T` and `Void`. As such, one must redefine
`f(x::Void) = nothing` for every new function, even though the semantics are
identical. The requirement for always defining such repetitive functions
therefore may be perceived as less safe because code that is tested only
against values of type `T` may fail in untested cases with a method error if
applied to a value of type `Void`. This could be resolved by static analysis
tools that check for totality.

### `Union{Some{T}, Void}` Pros & Cons

This approach is essentially the inverse of the `Union{T, Void}`: here no
methods will work on objects of this type unless they are defined for both
`f(x::Some{T})` and `f(x::Void)`. As such, this approach requires consistent
of specialized lifting machinery via dot-broadcasting syntax. See the later
section on lifting for details.

Because users will have to work with lifted functions, this approach is
less convenient, but more general and often safer. It also allows one to
distinguish between outcomes in cases where a method returns something like
`Union{Some{Void}, Void}` -- as might happen in a hash table implementation
that returns `Some` when a key is present and associated with `nothing` as a
value, but returns `Void` when a key is not present.

### `Union{T, Null{T}}` Pros & Cons

This approach allows us to distinguish `Null{Bool}` from `Null{Int}` in
cases in which that distinction might be useful. R makes such a distinction,
although it seems to be done only because it is falls out naturally from an
implementation that uses sentinel values. Otherwise, it is similar to the
`Union{T, Void}` case described above, except for one crucial problem.

That problem is essentially the problem we already have with `Nullable{T}`:
writing "clean" functions requires information about type-inference because of
a coupling of result types to knowledge about counterfactual outcomes in
branches.

To see the issue, consider the following possible implementation of
`map(f, x::Nullable{T})` in our current implementation of `Nullable{T}`:

```
function map(f, x::Nullable{T})
    if hasvalue(x)
        return Nullable(f(value(x)))
    else
        return Nullable{return_types(f, T)}()
    end
end
```

The problem is that, when given a null input, the return type can only match
the counterfactual return type that would have occurred if called on a non-null
value if the function knows the return type of `f`.

In static languages with arrow types this is not a problem as the return type
of a function is guaranteed to be well-defined by the semantics of the
language. But Julia does not provide such guarantees, so type-inference must
be invoked to determine information about the return type of the counterfactual
branch.

Note that this problem does not occur for the non-null case because we are
expected to evaluate `f`. It is only a problem when we need to know the return
type of `f`, but must not determine this type by evaluation of `f`.

## Step 2:: Define Core Operations on Nullable Types by Lifting

The core semantic issue for nullable types is maximizing code reuse: we want
to evaluate `f(x::?T)`, but not require that every function be defined for
both `f(x::T)` and `f(x::?T)` separately as the overwhelming majority of
functions will return `null` as their output whenever any of their arguments
are `null`. Implementing this lifting of functions can be solved as follows:

First, we define `broadcast(f, x::Nullable)` as follows:

```
immutable Lifted{T}
    func::T
end

lift(f) = Lifted(f)

broadcast(f, x::Nullable) = broadcast(lift(f), x)

broadcast(f′::Lifted, x::Nullable) = hasvalue(x) ? f′.func(value(x)) : null
```

This allows us to give default semantics to all functions that implement the
natural approach to lifting, while still making it possible to handle special
case functions that require distinct semantics.

Some such functions are:

* `hasvalue`
* `isnull`
* `get`
* `get_or_default`

There are natural extensions of this logic to the n-ary function case.

Because we would use broadcasting, lifting would happen via dot-syntax:
`x .+ y` would be lifted by default. In addition, we would special several
operators like `+`, `-`, `==`, etc. to automatically be equivalent to
broadcasting on nullables.

## Step 3: Introduce Syntactic Sugar for Nullable Types

Following the lead of
[Flow for Javascript](https://flowtype.org/docs/nullable-types.html), we
would introduce `?T` as sugar for `Nullable{T}`. We have also considered C#'s
`T?`. Which is chosen will likely depend upon how they interact with the
ternary operator.

We might also introduce a null-coalescing operator of `??` with
right-associativity that essentially performs lifting: in this case,
`x ?? y ?? 0` would be equivalent to `hasvalue(x) ? x : (hasvalue(y) ? y : 0)`.
