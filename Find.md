# JULEP find

- **Title:** Reorganize Search and Find API
- **Authors:** Milan Bouchet-Valat <<nalimilan@club.fr>>
- **Created:** December 10, 2016
- **Status:** work in progress

## Abstract

The current `find` and `search` families of functions are not very consistent with regard to
naming and supported features. This proposal aims to make the API more systematic. It is based
on ideas discussed in particular in [issue #10593](https://github.com/JuliaLang/julia/issues/10593)
and [issue #5664](https://github.com/JuliaLang/julia/issues/5664).

## Current Functions

Currently there are (at least) five families of search and find functions:
- `find` `findn` `findin` `findnz`, `findfirst` `findlast` `findprev` `findnext`
- `[r]search` `[r]searchindex` `searchsorted` `searchsortedlast` `searchsortedfirst`
- `match` `matchall` `eachmatch`
-  `indmin` `indmax` `findmin` `findmax`
- `indexin`

In the `find` family, `find` and `findn` return indices of non-zero or `true` values.
`findfirst`, `findlast`, `findprev` and `findnext` are very similar to `find`, but
iterative. `findin` allows looking for all elements of a collection inside another one.
Finally, `findnz` is even more different as it only works on matrices and returns a tuple
of vectors `(I,J,V)` for the row- and column-index and value.

In the `search` family, `[r]search` and `[r]searchindex` look for strings/chars/regex in a
string (though they also support bytes), the former returning a range, the latter the first
index. `searchsorted`, `searchsortedlast` and `searchsortedfirst` look for values equal to
or lower than an argument, and return a range for the first, and index for the two others.

The `match`, `matchall` and `eachmatch` functions deal with regular expressions. `match`
returns a special `RegexMatch` object with offsets and matches. `matchall` returns all
matching substrings. `eachmatch` returns an iterator over matches.

The `indmin` and `indmax` functions are quite different, as they return the index of the
minimum/maximum value. `findmin` and `findmax` return an `(index, value)` tuple of these
elements.

Finally, `indexin` is the same as `findin` (i.e. returns index of elements in a collection),
but it returns `0` for elements that were not found, instead of a shorter vector.

## Dimensions of Variation

This diversity can be organized along several dimensions, which are not always combined
systematically in the existing API:

- **Mode of operation**: look for all matches at once (`find`, `findin`, `indexin`),
iteratively forward (`findnext`, `search`), iteratively backwards (`findprev`, `rsearch`),
or look for the first match (`findfirst`, `searchsortedfirst`) or the last one
(`findlast`, `searchsortedlast`)

- **Look for**: non-zeros or `true` entries (`find(A)`), predicate-test-true
(`find(pred, A)`), elements present in a collection (`findin`, `indexin`),
elements equal to a value (`findfirst(A, v)`, `findlast(A, v)`, `findnext(A, v)`,
`findprev(A, v)`), extrema (`findmin`, `findmax`), or a range of elements matching
a sequence (`search*`, mostly for strings)

- **Return**: linear indices (most `find*` functions), cartesian indices (`findn`),
cartesian indices and values (`findnz`), range of linear indices (`search*`)

- **Return when not found**: shorter vector for all-at-once functions (`find`, `findin`,
`findn`, `findfnz`) except `indexin` (which includes a `0` entry), `0` for functions
returning a single index

## Summary of Current Status

The following table reorganizes existing methods which return linear indices based on the
first two dimensions described above:
- Whether to return all matches, only the next one, or only the previous one.
- What values to look for.

|  | nonzeros | test predicate `pred` | in collection `c` | equal to value `v` | sequence or regex `s` | extrema |
| --- | --- | --- | --- | --- | --- | --- |
| All at once | `find(A)` | `find(pred,A)` | `findin(A,c)` | `searchsorted(A,v)` | | `indmin(A)`/`indmax(A)` |
| Next match | `findnext(A,1)` | `findnext(pred,A,1)` | | `findnext(A,v,1)` | `search(A,s,1)` | |
| Previous match | `findprev(A,endof(A))` | `findprev(pred,A,endof(A))` | | `findprev(A,v,endof(A))` | `rsearch(A,s,endof(A))` | |

Some functions do not fit into this table:
- `findfirst` and `findlast` are special cases of `findnext` and `findprev`.
- `searchsortedfirst` and `searchsortedlast` give each a part of the result of `searchsorted`.
- `[r]searchindex` give part of the result from `[r]search`.
- `findn` and `findnz` do not return linear indices.
- `indexin` is similar to `findin` but returns `0` for entries with no match.
- `match`, `matchall` and `eachmatch` return `RegexMatch` objects or strings rather than
indices.
- `findmin` and `findmax` return both the index an value of extrema.

## Open Design Issues

- **How to switch between forward and backward search**:
    - Separate functions (e.g. starting with `r` for "reverse"): not great for
    documentation, harder to find, not very Julian.
    - `rev=false` positional argument: not very explicit.
    - `rev=false` keyword argument: clearer and consistent with `sort`, but maybe too slow
    (especially for single-element functions).
    - special object like `Order.Forward`/`Order.Backward`: clearer, but these objects do
    not have this meaning in Base, and introducing separate objects just for this may not
    be worth it.

- **Whether to keep the the `find`/`search` distinction**:
    - Obviously requires clearly distinct meanings for each family of functions.
    - Advantage: less complex signatures (due to many methods) for users, and limits
    dispatch ambiguities.
    - Drawback: using two different names for related functions makes it harder to find
    one variant when you know the other one; in particular, auto-completion does not help.

- **How to search iteratively**:
    - Functions returning the next/previous match after a given index: simple, but require
    manual handling of indices.
    - Functions returning an iterator over matches: more user-friendly, though overkill when
    you only want the next match.
    - The first approach can fit all needs (even if it can be cumbersome), but the second
    one can only replace the first one if it supports creating an iterator starting from
    a given index (on which you can call `first` to get the first match).


## General Proposal 1

The first proposal uses `find` for all-at-once variants, and `search` for iterative
variants. The variants returning iterators (last two rows) do not correspond to existing
functions: they could be added later, or never, without breaking the consistency of the API.
In this proposal, it is not possible to have both methods returning the next/previous match
and methods returning an iterator starting from a given index: the signatures would be the
same.

|  | nonzeros | predicate test | in collection `c` | equal to `v` | sequence or regex `s` | extrema |
| --- | --- | --- | --- | --- | --- | --- |
| All at once | `find(A)` | `find(pred,A)` | `findin(A,c)` | `findeq(A,v)` | `findseq(A,s)` | `findmin(A)`/`findmax(A)` |
| Next match | `search(A,1)` | `search(pred,A,1)` | * | `searcheq(A,v,1)` | `searchseq(A,s,1)` | |
| Previous match | `search(A,endof(A),true)` | `search(pred,A,endof(A),true)` | * | `searcheq(A,v,endof(A),true)` | `searchseq(A,s,endof(A),true)` | |
| Forward iterator | `search(A)` | `search(pred,A)` | * | `searcheq(A,v)` | `searchseq(A,s)` | |
| Backward iterator | `search(A,true)` | `search(pred,A,true)` | * | `searcheq(A,v,true)` | `searchseq(A,s,true)` | |

\* These combinations are not needed as they correspond to `searchseq`. Indeed they do not
exist in the current API.

## General Proposal 2

The second proposal uses `find` for functions returning one or several indices (either
all-at-once or iterative), and `search` for functions returning iterators (which
currently do not exist). Contrary to the first proposal, it therefore allows for
iterators starting at a a specific index. If those variants were not added in the end,
only `find` would exist. Conversely, methods returning the next/previous match could be
droppped in favor of iterators.

|  | nonzeros | predicate test | in collection `c` | equal to `v` | sequence or regex `s` | extrema |
| --- | --- | --- | --- | --- | --- | --- |
| All at once | `find(A)` | `find(pred,A)` | `findin(A,c)` | `findeq(A,v)` | `findseq(A,s)` | `findmin(A)`/`findmax(A)` |
| Next match | `find(A,1)` | `find(pred,A,1)` | * | `findeq(A,v,1)` | `findseq(A,s,1)` | |
| Previous match | `find(A,endof(A),true)` | `find(pred,A,endof(A),true)` | * | `findeq(A,v,endof(A),true)` | `findseq(A,s,endof(A),true)` | |
| Forward iterator | `search(A)` | `search(pred,A)` | * | `searcheq(A,v)` | `searchseq(A,s)` | |
| Backward iterator | `search(A,true)` | `search(pred,A,true)` | * | `searcheq(A,v,true)` | `searchseq(A,s,true)` | |

\* These combinations are not needed as they correspond to `searchseq`. Indeed they do not
exist in the current API.

## Particular Cases

Other issues are more localized and can be fixed one by one, depending on the chosen general
plan.

- **`findin`**: This function is just a shorthand for `find(x -> x in c, A)`. It may not be
needed now that anonymous functions are fast.

- **`findmin` and `findmax`**: `findmin` and `findmax` are inconsistent
with both proposals, since they return an `(index, value)` tuple instead of an index. They
should be changed to return an index (as in both proposals above). A new name needs to be
found if we want to keep `(index, value)` variants, which are slightly more efficient.

- **`searchsorted*` functions**: These functions should be replaced with
standard search/find functions called on a special `SortedArray` wrapper. `findeq` would
replace `searchsorted` and -- like that function -- return a range (instead of a `Vector`)
of indices, which is possible when input is sorted.

- **`indexin`**: It is not clear whether this function really belongs to the
find/search family. It could be kept as-is.

- **`*match*` functions**: These functions (`match`, `matchall` and `eachmatch`)
return `RegexMatch` objects or strings (rather than indices). They can be left outside the
scope of this Julep. On the other hand `findseq`/`searchseq` functions should support regexes
for consistency, only returning ranges of indices (as does `search` currently).

## Deprecation strategy

Depending on the choices made, the migration to the new API will be possible in a single
release (if no ambiguity exists with the old one), or it will have to be done in two
releases (to allow removing old conflicting methods first).

## Issues Beyond the Scope of This Julep

These are important to resolve but are not covered by the above proposals.

- **Whether to return a `Nullable` instead of `0` when there is no match**
([PR#15755](https://github.com/JuliaLang/julia/pull/15755)): This is blocked by progress
with regard to `Nullable`, in particular whether they are stack-allocated in all cases
and whether they can be represented as a `Union` type. It is therefore out of this Julep's
scope.

- **Whether to return linear or cartesian indices**
([PR#14086](https://github.com/JuliaLang/julia/pull/14086)): Both could be needed depending
on the context. Passing `CartesianIndex` as the first argument to all functions would work
and would allow replacing `findn(A)` with `find(CartesianIndex, A)`. On the other hand,
computing the linear index is slow for `LinearSlow` arrays, which means that returning
the same index type as `eachindex(A)` could be a better default; it also makes more sense
for multidimensional arrays. Then one would write `find(LinearIndex, A)` or  `find(Int, A)`
to always get a linear index.

- **Sentinel values in a world where array indices do not necessarily start with 1**:
    - `findfirst(x, v)` returns 0 if no value matching `v` is found;
      however, if `x` allows 0 as an index, the meaning of 0 is
      ambiguous. One could return `typemin(Int)` or
      `minimum(linearindices(x))-1`, but what if `x` starts indexing
      at `typemin(Int)`?
    - No matter sentinel value gets returned, the deprecation
      strategy here is delicate. There may be a lot of code that
      checks the return value and compares it to 0.
