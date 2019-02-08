# Julep: Internal Properties

## The problem

The julia syntax `x.a` can refer to either a public or internal property `a` of `x`; it is not possible to know from a *local* reading of the syntax. This makes it hard to detect violation of API boundaries in code review and almost impossible with a linter.

There are two sub cases of this syntactic problem:
* The fields of a type may be public or internal, for example, `.re` and `.im` fields of `Base.Complex`. Access to internal fields may be restricted by overriding `getproperty`, but this solution leaves module implementors stuck with ugly syntax.
* The bindings in a module may be public or internal. Public but non-exported bindings are common in the ecosystem.

## Goals

This Julep aims to resolve this problem by making the violation of API boundaries syntactically obvious while respecting Julia's public-by-default aesthetic. Specific goals:

1. The use of internal properties should be syntactically obvious outside the originating module.
2. The declaration of internal properties should be syntactically clear.
3. `getfield` should be easily accessible with natural syntax in the originating module.
4. Avoid adding syntax and consuming precious ascii characters if possible.
5. Reuse existing naming conventions in the ecosystem
6. Retain the public-by-default aesthetic where the implementation is in easy reach from the right places.

## Existing conventions and prior discussion

Many packages in the ecosystem prefix module-internal bindings with an underscore. However, doing this for internal fields of `struct`s is much rarer. Some packages use `getproperty` to add convenient access to computed properties of concrete types. I'm not aware of cases where getproperty is used for access control.

### Prior discussion

* See extended discussion of binary `$` as notation for `getfield` over at https://github.com/JuliaLang/julia/issues/25750.
* Binary `@` also appears to be available and was suggested in the discussion of https://github.com/JuliaLang/julia/pull/30646; some of the somewhat off topic musings there led to this julep.
* The meaning of `export` with regard to public symbols has been discussed in many places.
