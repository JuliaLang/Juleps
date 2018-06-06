# JULEP string formatting

- **Title**: Julia string formatting
- **Author**: Simon Byrne <simonbyrne@gmail.com>
- **Created**: Feb 2018
- **Status**: WIP

## Background

Base Julia currently provides 2 distinct ways to substitute variables into strings

- [`$` string interpolation](https://docs.julialang.org/en/stable/manual/strings/#string-interpolation-1): this internally calls the `print`/`string` functions, which for numbers use the GRISU module for printing in the "shortest roundtrip" format. Though internally a few options are implemented (e.g. shortened forms used for printing arrays) none of these are currently exported, other than via the `:compact` global IO option.
 
- [`@printf`](https://docs.julialang.org/en/stable/stdlib/io-network/#Base.Printf.@printf)/[`@sprintf`](https://docs.julialang.org/en/stable/stdlib/io-network/#Base.Printf.@sprintf) macros, which provide C-style `printf` behaviour. This has several downsides:
  - requires users to know all the different format specifiers
  - limited functionality (e.g. cannot display shortest roundtrip format used by `print`/`string`)
  - options are hard-coded at macro expansion time
  - positional references make it cumbersome to use
  - not easily extensible to new types

Some additional functionality is also provided in third party packages:
- [Formatting.jl](https://github.com/JuliaIO/Formatting.jl)
- [StringLiterals.jl](https://github.com/JuliaString/StringLiterals.jl)

## Summary

There are 3 primary theses of this Julep:

1. Variables should use direct substitution
2. Formats should be specified using Julia syntax and not a distinct mini-language
3. It should cover existing use cases, and be extensible to new ones.

### Method of substitution

There are broadly 3 ways of doing substitution (see Other Languages):

 1. **positional** in which variables are provided as additional positional arguments. This includes traditional `printf` style, in which substitutions occur in the order they are provided, as well as numbered substations.
 2. **keyword** in which substitutions are given keywords, and then provided as keyword arguments.
 3. **direct** in which variables or expressions appear directly in the string.

I contend that options positional and keyword substitution are poor choices, primarily in that they add an unnecessary layer of indirection. Their main advantage seems to be the ease with which they can be implemented in compiled languages (C# and Swift being the only compiled language with direct substitution). Moreover, since direct substitution is already supported (and widely used) in Julia, it makes sense to extend it to support formatting.

### Format specification

All other languages seem to rely on various mini-languages which use a different syntax (the exception being Common Lisp, for which arguably the mini- prefix does not apply). This has several disadvantages:

 - it is yet another syntax for users to learn (additionally, since it is used less frequently, there is the extra cost of re-learning each time).
 - moreover, the syntax is generally poor: they rely on single character modifiers (which often lack useful mnemonics), which can do different things in different circumstances (e.g. the `#` flag in `printf`).
 - they are often not extensible: one of Julia's primary features is the ease with which users can define new types for their own needs. Similarly, there should be an easy way to support formatting these types.
 - they encourage copy-paste repetition (if I want to print out several variables to 3 decimal places, I have to copy-paste `%.3f` around in a bunch of places).

The only real advantage is the `printf` format is often somewhat consistent between languages (e.g. `%.3f`), but even then certain features can differ between implementations.

Instead I propose that we make use of Julia syntax wherever possible. There are several non-breaking ways to do this with existing `$` interpolation.

### Support all existing uses and be extensible to new cases

We want to support all existing functionality, including:
- fixed number of fractional digits (`"%f"`)
- fixed number of significant digits (`"%e"`/`"%g"`)
- shortest roundtrip (`print`/`string`)
  - this is the shortest decimal representation which will be converted back to the input number
- shortest roundtrip, bounded number of digits (not exposed, but used in array printing)
  - actually a common question on StackOverflow, but no other language provides it
- hex-float (`"%a"`)

While this is primarily concerned with the printing of text, integers and floating point numbers, this should also be extensible to other types (e.g. rationals, dates, unitful quantities, etc.) 

### Additional 

Additional things I would like to support:
- base-2 floats ([#9371](https://github.com/JuliaLang/julia/issues/9371))
- print exact value to full decimal precision
  * i.e. `0.1` => `"0.1000000000000000055511151231257827021181583404541015625"`
- custom radix point
- custom exponent symbol (needed for current `Float32` printing)
- option for no leading 0 (e.g. `0.1` as ".1")
- digit separators (both for left and right of radix point)
  * typically 3 digits at a time (i.e. thousands), but grouping should be customisable
  * See https://blogs.msdn.microsoft.com/oldnewthing/20060417-06/?p=31513
- localisation support
- custom prefix between sign and number
  * e.g. "0x", "$"
- manual control of when exponent notation is used
- print negatives with parentheses
- "Engineering" style
  * exponent as integer multiple of 3
- rounding control: print to 2 decimal places, rounding up
- percentages without intermediate rounding
- custom character sets (for both significand and exponent)
- alignment control
- control printing of nested objects (e.g. print an array to full precision)

## Other languages
A quick summary of other languages

- [`printf`](https://en.wikipedia.org/wiki/Printf_format_string), used in many languages, most notably C.
- Python currently provides 4 methods for formatting strings:
  * [% substitution](https://docs.python.org/3/library/stdtypes.html#printf-style-string-formatting): uses printf style formats
  * [`str.format`](https://docs.python.org/3/library/stdtypes.html#str.format)/[`str.format_map`](https://docs.python.org/3/library/stdtypes.html#str.format_map): uses its own [format string syntax](https://docs.python.org/3/library/string.html#formatstrings), based on positional/named references.
  * [Template strings](https://docs.python.org/3/library/string.html#template-strings): substitution using `$`, does not provide any formatting options
  * [f-strings](https://docs.python.org/3/reference/lexical_analysis.html#f-strings)): uses the same syntax as `str.format`, but allows expressions to be substituted directly
- JavaScript uses backticks to denote template strings which support direct expression substitution. No special formatting syntax is provided.
- C++ does funny things with [`<<`](http://www.cplusplus.com/reference/ostream/ostream/operator%3C%3C/). There are also various 3rd party libraries
  * [fastformat](http://www.fastformat.org/)
  * [fmtlib](https://github.com/fmtlib/fmt)
- Rust [std::fmt](https://doc.rust-lang.org/std/fmt/) uses positional and/or keyword arguments, with its own minilanguage (appears similar to Python's).
- Haskell [`printf`](https://hackage.haskell.org/package/base-4.10.1.0/docs/Text-Printf.html) uses positional substitution, and usual printf formatting
- C# has 2 formats
  * [`String.Format`](https://msdn.microsoft.com/en-us/library/system.string.format(v=vs.110).aspx) uses positional substitution
  * [Interpolated strings](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/interpolated-strings) provide direct substitution, with its own minilanguage for formatting.
- Common lisp [`Format`](https://en.wikipedia.org/wiki/Format_(Common_Lisp)) uses positional substitution, but its minilanguage provides support for things like iteration and recursion (and is quite possibly Turing complete).
- Swift has
  - [`String`](https://developer.apple.com/documentation/swift/string) supports positional substitution with `printf`-style formatting
  - [direct substitution](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/StringsAndCharacters.html#//apple_ref/doc/uid/TP40014097-CH7-ID292), but no formatting (suggestions to get around this rely on [calling the former inside the latter](http://en.swifter.tips/output-format/)).

## Proposal

The general proposal is to specify the format via additional arguments to interpolated strings. This can utilise either keyword or positional arguments (or no arguments for default printing):

    "π is approximately $pi."
    "π is approximately $(pi, fracdigits=4)."
    "π is approximately $(pi, Decimal(4))."

The multiple-argument ones are currently syntax errors, so will not present backward compatibility problems. The intention is that keyword arguments expose the straightforward features, and more complicated features and extensions can be provided by Julia objects passed as second arguments.

### Implementation

Interpolation would call a function `stringformat`. The above would each call something like

    string("π is approximately ", stringformat(pi), ".")
    string("π is approximately ", stringformat(pi, fracdigits=4), ".")
    string("π is approximately ", stringformat(pi, Decimal(4)), ".")    

where `stringformat` is the appropriately defined function returning a formatted string.

##  Performance considerations

When considering performance, it is worth considering the desired result.

### Output to string

The user wants to output a Julia `String` object, similar to current `@sprintf` behaviour. In this case, we want to avoid the intermediate allocations of the results of `stringformat`. Ideally we would want this to lower to something like

    iobuf = IOBuffer()
    print(iobuf, "π is approximately ")
    printformat(iobuf, pi, fracdigits=4)
    print(iobuf, ".")
    String(take!(iobuf))

where `printformat` instead writes the formatted string directly to `io`.

### Output to IO

The user wants to output directly to an IO device (say `IOStream` or `IOBuffer`), similar to current `@printf` behaviour. In this case, we want to avoid allocating any string at all, i.e. we want to lower 

    print(io, "π is approximately ", stringformat(pi, fracdigits=4), ".")

directly to

    print(io, "π is approximately ")
    printformat(io, pi, fracdigits=4)
    print(io, ".")

If this is not possible, we could provide a macro `@print` which would do the necessary rewriting, e.g.

    @print(io, "π is approximately $(pi, fracdigits=4).")

