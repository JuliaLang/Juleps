## The problem

Julia APIs use a mixture of public and internal fields so it's never syntactically obvious whether a given field access `x.a` violates API boundaries. This makes it hard to detect these problems when reviewing code and hard to write a linter which can tell the difference. That is, a *local* reading of the code can't tell the story; you must read the documentation of whatever package you're using.

## Goals

This Julep aims to make the access of internal fields syntactically obvious.  Ideally this is just ugly enough to make you think twice but not so bad that it becomes difficult to use the internal structure of types inside internal functions.  Some specific goals:

1. Accessing internal fields should be syntactically obvious.
2. Declaring internal fields in structs should be syntactically obvious and consistent with the notation for access.
3. For use in the originating module it would be nice to have short syntax for `getfield`.
4. Avoid adding syntax and consuming precious ascii characters if possible.
5. Respect existing naming conventions in the ecosystem and aesthetic qualities of the language.

## Prior discussion

* See extended discussion of binary `$` as notation for `getfield` over at https://github.com/JuliaLang/julia/issues/25750.
* Binary `@` also appears to be available and was suggested in the discussion of https://github.com/JuliaLang/julia/pull/30646; some of the somewhat off topic musings there led to this julep.
* The meaning of `export` with regard to public symbols has been discussed in many places.
