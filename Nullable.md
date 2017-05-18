# JULEP Nullable

- **Title:** Nullable Redesign
- **Authors:** John Myles White <<jmw@johnmyleswhite.com>>
- **Created:** December 9, 2016
- **Status:** work in progress

## Abstract

The representation of missing values in Julia is too complicated. We currently
have at least three kinds of null-like constructs:

* A `Void` type with a singleton value `nothing`, which is the return value of
    functions that don't return another kind of value;
* A `Nullable{T}` parametric type with many values, which allows programmers
    to represent the possibility of missing/invalid values;
* An `#undef` quasi-value, which reflects the use of null pointers in Julia's
    implementation and is exposed after allocating memory without
    initializing that memory.

This Julep describes a proposal to merge these three constructs into one
construct.

In particular, the proposal is to replace all existing null-like constructs
with a redesigned version of the current `Nullable` type. This new version of
`Nullable` will involve both (a) changing the internal implementation of the
current `Nullable` type and (b) clarifying the semantics of the `Nullable`
type. Because the name `Nullable{T}` is a little long, we will use the
notation, `?T`, as a short-hand for `Nullable{T}` throughout this document.
Making this short-hand a genuine part of Julia's syntax is one part of this
Julep's proposed design.

## Overview of Goals

The core goals for this Julep are to:

* Simplify the language by replacing many similar null-like constructs with a
    single null-like construct.
* Ensure that the resulting representation of `null` is a first-class citizen
    of the language. In particular, we will remove the second-class citizen
    status of `#undef`, which is a quasi-value that can be present in
    uninitialized data structures, but cannot be inserted into those data
    structures.
* Clarify how functions can be applied to nullable types by formalizing an
    extensible protocol for applying methods defined over `T` to values of
    type `?T`.
* Make it possible to write optimized code for nullable types without having
    to call `Core.Inference.return_type` or `promote_type`.
* Introduce syntactic sugar for nullable types and also introduce special
    binary operators for working with nullable-typed objects.

The optional goals for this Julep are to:

* Replace the `NullableArray{T}` type with `Array{?T}`, while maintaining
    the ability to pass the raw masked values as a single block of memory
    through Julia's FFI in the way that `NullableArray{T}` currently allows
    passing the `values` field without passing the `isnull` mask.

The prerequisites for this Julep are to:

* Complete optimizations to the Julia compiler to handle `Union{T, Void}` as
    efficiently as our current `Nullable` type handles the union of the set of
    non-null values `Nullable{T}(val)` and the singleton set containing the
    null value `Nullable{T}()`. These compiler changes may include memory
    layout changes as well as optimization of run-time dispatch among
    pre-specified types via low-cost branches.

## Step 1: Redesigning the Nullable Type's Internal Implementation

Assuming that compiler optimizations are put in place for union types, the
first step for this Julep is to replace the concrete type `Nullable{T}` with an
optimized non-concrete union type. The options for replacing `Nullable{T}`
that have been proposed are the following union types:

* `Union{T, Void}`
* `Union{Some{T}, Void}`
* `Union{T, Null{T}}`

Any newly optimized union type would have the same semantics as union types
currently have: in particular, a union type is not a concrete type and, as
a consequence, there are no values of any non-trivial union type. The use of
a non-concrete type is important both for optimization purposes and for
clarifying the possible semantics we can define for the new nullable types,
as we would be replacing a concrete type with a non-concrete type.

Deciding between these three options will require balancing a variety of
concerns. The arguments for each option are summarized below. We include all
of them for clarity, although we believe that the arguments on behalf of
`Union{Some{T}, Void}` are the strongest.

### `Union{T, Void}` Pros & Cons

When a function returns `Union{T, Void}`, the user can immediately apply all
methods defined on `T` when the value happens to be of type `T`. Allowing users
to call existing methods without having to unwrap non-null values can be
convenient.

In this regard, the use of a `Union{T, NAtype}` pattern in DataArrays was
perceived by many to offer greater convenience than our current `Nullable{T}`
type provides, although the use of a union type was also the cause of that
library's historical poor performance because of the compiler's non-optimized
handling of union types. This Julep assumes that union types will be optimized
such that poor performance is no longer a consequence of using a union type as
the implementation of `?T`.

Instead of concerns about inferior performance, one major argument against
using `Union{T, Void}` is that a `Union{T, Void}` type can only be used
reliably in code if the methods being called on instances of the union type
are defined for both the `T` and `Void` sub-types of that union. Going forward,
this document will refer to these two cases as the non-null and null cases,
respectively.

Because the `Union{T, Void}` representation uses a union type, it is possible
for a method to be defined only over a subset of the values in the union. In
particular, the possibility of directly calling `f(x::T)` on a value of type
`Union{T, Void}` means that it is possible for code to handle the non-null case
without being capable of handling the null case. We will refer to situations in
which code handles only one of the non-null or null cases without handling the
other case as failing to exhibit totality, a term used because the code does
not define an execution path for the totality of all possible inputs.

To ensure totality, users must either (a) use functions for which both
`f(x::T)` and `f(x::Void)` are defined or (b) use higher-order functions
that can take in a function with a method of the form `f(x::T)` and employ a
default semantics for null values to handle the `f(x::Void)` case. In
particular, we will refer to the pattern of extending an `f(x::T)` method to
handle `f(x::Union{T, Void)` by assuming that `f(x::Void) = nothing` as
lifting.

The unique concern with `Union{T, Void}` is `f(x::T)` will often be defined
because it is exists even in the parts of Julia that deal with only
non-nullable types. When `T` is `Int`, this path is any function on integers.
Many such functions exist and it is possible that users will assume that
`f(x::Void)` will also exist because `f(x::Int)` exists.

In DataArrays, the closely related problem of needing definitions of
`f(x::NAtype) = NA` for all functions `f` was addressed by manually defining
`f(x::NAtype) = NA` for all functions `f` that commonly occurred in practice.
Although this piece-meal approach worked effectively in many cases because the
majority of real code makes use of only a small number of core functions,
there was always yet another function that to had be defined manually when a
user found that `f(x::NAtype)` was not defined for their favorite function `f`.
In this sense, the exact problems introduced by manually vectorizing scalar
functions in Base Julia were repeated again when adding methods for `NAtype`:
instead of describing a universal pattern using a generic abstraction, we
repeated that pattern manually over and over again, while always discovering
new cases that had been left out.

In addition to requiring repetitive code to provide uniform semantics for
`f(x::Void)`, there are concerns that the convenience of not having to unwrap
values may encourage unsafe practices that will not be caught by shallow testing
practices. For example, if a user defines `f(x::T)`, but not `f(x::Void)`,
and they only test their code against values of type `T`, they may release
code that they believe will work for `Union{T, Void}`, but will actually
raise a `MethodError` at run-time if `nothing` is ever encountered.

That said, if we adopted `typealias Nullable{T} Union{T, Void}` as our new
implementation of `Nullable{T}`, such concerns about safety could be addressed
by static analysis tools that check that null values are always handled
whenever they are allowed by a method's type signature. We might use something
like the approach taken in
[Flow for Javascript](https://flowtype.org/docs/nullable-types.html).

Finally, the use of `Union{T, Void}` introduces issues with method specificity
and the related issue of replacing run-time dispatch with compile-time
dispatch. In particular, the core problem is that `T <: Union{T, Void}`, so the
types are not disjoint, leading to specificity conflicts.

To see how the lack of disjointness can introduce problems, imagine that a user
defines the following two functions:

```
f(x::Int) = 1
f(x::Union{Int, Void}) = 2
```

In this case, the second method, which provides a definition for values of type
`Int`, will actually never be called on any values of type `Int` because the
second method definition is strictly less specific than the first method
definition for values of type `Int`. This can even lead to universally dead
paths in code that are not obvious at first glance:

```
f(x::Int) = 1
f(x::Union{Int, Void}) = isa(x, Void) ? 2 : 3
```

Here the second part of the branch will never be taken, but the unreachability
of this branch can only be determined by non-local analysis as it is the
existence of the method, `f(x::Int) = 1`, that renders a branch in other code
unreachable. Such dead paths are expected in large Julia codebases and can be
addressed by static analysis tools, but it may be problematic that this
representation of `?T` could increase the frequency with which these dead paths
are defined by accident.

As such, the `Union{T, Void}` approach suggests that one would only need to
define methods for `f(x::Int)` and `f(x::Void)` in practice, rather than
the union type. Because the `f(x::Void)` definitions would be trivial in
many cases, we would still need to use general approaches for lifting.

### `Union{Some{T}, Void}` Pros & Cons

This approach involves wrapping the non-null side of a nullable in a `Some`
wrapper that allows one to distinguish `T` from `Some{T}`. This small amount of
indirection has many implications because it ensures that
`T <: Union{Some{T}, Void}` is not true, breaking many of the patterns we noted
with `Union{T, Void}`.

For example, because a wrapper is employed, no functions with methods defined
on `T` will work on objects of type `Union{Some{T}, Void}` unless those methods
are defined for `f(x::Some{T})` and `f(x::Void)`. For example, the existence of
`f(x::Int)` does not imply that `f(x::Some{Int})` or `f(x::Void)` will be
defined.

This limitation can also be framed as a benefit because it ensures that
multiple methods can co-exist without the specificity concerns raised for
`Union{T, Void}`.

For example, it would be possible to write code that hits all branches of the
following method definitions:

```
f(x::Int) = 1
f(x::Union{Some{Int}, Void}) = isa(x, Void) ? 2 : 3
```

To repeat this point in other terms,
the use of `Some{T}` instead of `T` provides a way of having the `T` and `?T`
types co-exist without specificity issues, but it simultaneously ensures
that no existing methods for `T` automatically apply to `?T`. It provides
arguably cleaner semantics (by making `T` and `?T` disjoint) at the cost of
making the non-null case for nullable types less convenient to work with.

As such, this implementation of `?T` would require that we push users towards
the consistent usage of specialized lifting machinery that knows how to handle
`f(x::Union{Some{T}, Void})` given only knowledge of how `f(x::T)` works. For
one approach to this problem, see the later section on lifting functions from
non-nullable types to nullable types.

Because users will have to work with lifted functions, this approach makes it
more difficult for users to write code that works for the non-null case, but
does not work for the null case. But the possibility of defining methods for
only one of `Some{T}` or `Void` means that a totality checker would still be
needed because users can always define only one of `f(x::Some{T})` and
`f(x::Void)`. (We should note that the current implementation of `Nullable{T}`
also does not ensure totality.)

Another point in favor of `Some{T}` is that it is sometimes useful to
distinguish between two confusable outcomes in cases where a method returns
something like `Union{Some{Void}, Void}` -- as might happen in a hash table
implementation that returns `Some(nothing)` when a key is present and
associated with `nothing` as a value, but returns `nothing` when a key is not
present. This is another implication of working with disjoint types for `T` and
`?T`.

On the other hand, the use of `Some{T}` raises issues about how we expect `T`,
`Some{T}` and `Void` to interact. For example, consider the following code:

```
function sum(xs::Vector{Union{Some{Int}, Void}})
    s = 0
    for x in xs
        s += x
    end
    return x
end
```

What type does this return when `xs = ?Int[]`? What type does it return when
`xs = ?Int[Some(1)]`? What type does it return when `xs = ?Int[nothing]`?

At a minimum, the use of `Some{T}` would require careful thought about
type conversion and type promotion rules since we would need to define methods
for combinations like `+(x::Int, y::Some{T})`. Almost surely, we would need to
allow promoting `T` to `Some{T}` to make common code patterns work without
much verbosity.

### `Union{T, Null{T}}` Pros & Cons

This final approach allows us to distinguish `Null{Bool}` from `Null{Int}` in
cases in which that distinction might be useful. R makes such a distinction,
although it seems to be done only because it is falls out naturally from an
implementation that uses distinct sentinel values for each nullable type in
the language. Otherwise, this approach is similar to the `Union{T, Void}` case
described above, except for one crucial problem that would seem to suggest this
approach offers few benefits over our current implementation of `Nullable{T}`.

That problem is essentially the problem we already have with `Nullable{T}`:
writing functions that return concrete types as output generally requires
information about the return types of any functions used inside of a function
because one needs to know the return type associated with the non-null case
even when writing code to handle the null case. Needing to know about the code
path not taken will be referred to the "counterfactual return type problem" in
what follows.

To see how this issue arises in practice, consider the following possible
implementation of `map(f, x::Nullable{T})` given our current implementation of
`Nullable{T}`:

```
function map(f, x::Nullable{T})
    if !isnull(x)
        return Nullable(f(get(x)))
    else
        return Nullable{Core.Inference.return_type(f, (T, ))}()
    end
end
```

For example, ensuring that `map(sqrt, Nullable{Int})` always returns
`Nullable{Float64}` requires that the null path in the code be aware of the
return type of `sqrt(x::Int)`. This information is not generally available in
Julia.

In static languages with arrow types, the counterfactual return type problem is
not a problem at all because the return type of a function is guaranteed to be
well-defined by the semantics of the language and guaranteed to be knowable at
compile-time, so one can refer to properties of `f` without having to call into
the implementation details of the language. But Julia does not provide such
guarantees, so type-inference must be invoked to determine information about
the return type of the counterfactual branch. An alternative approach is to
use `promote_type` to explicitly register return types, but this is clearly
problematic as it essentially duplicates a function's declared return type,
but does so using a mechanism that can be inaccurate.

Note that this problem does not generally occur for the non-null case because
we are expected to evaluate `f` in that scenario. It is only a problem when we
need to know the return type of `f`, but must not determine this type by
any actual evaluation of `f`. Because this problem does not occur in the
non-null case, it does arise for either `Union{T, Void}` or
`Union{Some{T}, Void}`. The problem is unique to `Union{T, Null{T}}` and our
current `Nullable{T}` implementation as one must determine the type parameter
`T` even for the null case.

## Step 2:: Define Core Operations on Nullable Types by Lifting

Once an implementation of `Nullable{T}` is agreed upon, the core semantic issue
for nullable types is maximizing code reuse by making lifting easy to use. In
other words, we want to make it easy to evaluate `f(x::Nullable{T})` without
requiring users to define methods for every function to handle the null case,
for which the semantics are so consistent. We want to avoid requiring that
users write code for the null case because we know that the overwhelming
majority of functions will return `null` as their output whenever any of
their arguments are `null`.

As mentioned earlier, extending the definition of functions from `T` to `?T`
is called lifting. We propose the following extensible approach to implementing
lifting that defaults to the semantics we expect will occur most commonly, but
allows implementing alternative semantics where needed.

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

* `isnull(x::?T)`
* `get(x::?T)`
* `get(x::?T, default::T)`

Because we would use broadcasting, lifting would happen via dot-syntax:
for example, `x .+ y` would be lifted by default. In addition, we would special
several operators like `+`, `-`, `==`, etc. to automatically be equivalent to
broadcasting on nullables.

## Step 3: Introduce Syntactic Sugar for Nullable Types

Following the lead of
[Flow for Javascript](https://flowtype.org/docs/nullable-types.html), we
would introduce `?T` as sugar for `Nullable{T}`. We have also considered C#'s
`T?`. Which is chosen will likely depend upon how they interact with the
ternary operator.

We might also introduce a null-coalescing operator, `??`, with
right-associativity that essentially performs lifting: for example,
`x ?? y ?? 0` would be equivalent to `hasvalue(x) ? x : (hasvalue(y) ? y : 0)`.

Finally, we might introduce `.?` for lifted field access such that
`x?.field_name` would be equivalent to `getfield.(x, :field_name)`.

## Step 4: All Special Operators

There are several special operators that need to be defined. In particular,
we want to distinguish between non-short-circuiting Booleans and
short-circuiting Booleans. Following C#'s lead, we will not allow
short-circuiting Boolean operators to be applied to nullable types.

## Step 5: Memory Initialization and Nullables

At present, `#undef` occurs whenever memory is allocated for mutable types
without initializing that memory. Thus in the current version of Julia, the
following occurs:

```
julia> Array(String, 1)
1-element Array{String,1}:
 #undef
```

Our proposal would remove this behavior entirely.

Instead, there would be two possibilities:

* `Array{String}` can never contain `#undef` values and therefore must be
    initialized whenever it is allocated.
* `Array{?String}` is always initialized by the mere act of allocation, as
    the implementation will default to setting the value of `null` in all
    positions in the array.

This would likely occur concurrently with an attempt to use `calloc`
everywhere instead of `malloc`.

## Step 6: Optimizations

When working with nullables, we generally need to check a mask before we
can decide the value of an operation on nullable typed objects. This check
can sometimes induce a branch that is costly relative to the cost of the core
operation being performed (e.g. addition) and the existence of the branch may
also impede compiler optimizations such as automatic SIMD vectorization.

As such, we will want to ensure that `lift(f, args)` is customized to use
`ifelse` vs `if` for some functions like addition.

## Extended Usage Examples

* `mean(v::Array{?Int})`
* `median(v::Array{?Int})`
* `pdf(Binomial(n, 0.5), x)` where `n::?Int` and `x::?Int`
