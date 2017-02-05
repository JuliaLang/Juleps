# JULEP PathTypes

- **Title:** Using types for paths. 
- **Authors:** 
- **Created:** 31 Janurary 2017 (initial draft 2016-09-11)
- **Status:** work in progress

## Abstract:

Add a AbstractPath type  and deprecate  `open(::AbstractString)` infavour of `open(::AbstractPath)`
AbstractPaths allow code to be written without caring where or how the data is stored.
Using types for paths allow us to enfornce some validity and constancy rules.
This also allows for multiple dispatch differenciating between a Path to a file, and that files contents as a string.

## Proposal

I was tempted to name this: “Taking Paths Seriously”, but I resisted.
Though, that is what it comes down to in the end.

Julia does not have a Path type, to represent things like Filepaths.
Instead it currently takes the approach of representing them with Strings.
Which is not entirely awful, but has problems.

If in Julia you see a method like `foo_carrot(::Food)` and another `foo(::Food)`, then that is often as sign that `Carrot` should be a different type from `Food`, and that you should use multiple dispatch. We have this state with `join_path` and `join`.

More generally I often find myself wanting to define methods that take a:

`AbstractString` -- process the data that is in a String
`IO` -- Process the data from a stream
`AbstractString` (Filepath) --  that load data frome a file then process the contents

This to me is a pretty standard pattern -- to the extent I have started to define macros to define the various methods in terms of cannonical method.
But to allow it to work I need to define at least 2 functions, a `load(path::AbstractString)` and `load(io::IO)` and then `load_content(data::AbstractString)`.
So this got me thinking: I wish Path's were their own distinct type, so that I coud dispatch on them.



After thinking about [this for far, far, too long](http://white.ucc.asn.au/2016/09/14/an-algebraic-structure-for-path-schema-take2.html)
I conclude that a Path, can be defined at an highly abstract level as a hierarchical index, onto a directed multigraph. Noting that “flat” sets, trees, and directed graphs are all particular types of directed multigraphs. With a few certain properties.
And that once you have defined these couple of features, you can get a bundle of functionality “for free”. You can read the [blog post for details on that.](http://white.ucc.asn.au/2016/09/14/an-algebraic-structure-for-path-schema-take2.html). The  key takeaway, is that **a path is reference (i.e. an index), and `open` resolves (dereferences) that reference into the data it is pointing at.**; and that so long as the reference follows a few rules, we don't really need to care about more.

I suggest that julia should have an `AbstractPath` type, that represents an hierarchical index for data that can be `open`ed. Much like a Symbol is a type that represents the name, an path is a type that represents a location.
Abstract Path need not only represent a filepath, but any path more generally.
A URL, A Module/Submodule name, a XPATH expression, a Glob.

Each implementation of `AbstractPath`, would define a `RelativePath` type, a few methods for manipulating it generally, and 1 or more *Roots*, from which we would use a macro to define a `AbsolutePath`, as a root, paired with a `RelativePath`. Beyond this implementations could specify special methods that only work for that particular type of path.

The only method `AbstractPath` really needs to implement is a concatenate operation, that joins `RelativePath`, to an `AbstractPath`. And a `open` method that opens the location, if it exists.
Having a `parent` method also helps, once you have that, then you start to get all the free methods I mentioned earlier.

I suggest that Base would only contain implementation for Filepaths, and leave implementing other kinds of paths to packages.
As part of this implentation, a String macro should be defined to convert String literal into paths. I suggest `@p_str`, so that `p"C:/Program Files/julia/julia.exe"`.
CommonLisp is one of the only other languages with String Macros and Multiple dispatch. I believe they do something similar to this.

Having the path's as a type not only lets us allow weird, wacky, and useful paths; it also lets us bring sensible and robust handling based on being able to define different rules than are defined for strings. 

---

I’ve  specified distinguishing Absolute from Relative Paths.
This defination makes sense for all abstract paths.
For file paths, which are better explored, other libraries
show how it can also be broken up in other, additional  ways.


Many systems distinguish WindowsPaths from PosixPaths.
This makes sense since there any many differences. 
Like there being many differnt allowed roots for Windows paths; a different prefered joining slash; and case insensitivity.

[Python3's PathLib](https://docs.python.org/3/library/pathlib.html) also distiguishes, Pure Paths from Concrete Paths.
Pure Paths can operate without touching the filesystem, Concrete Paths do not make that promise.
Many operations on Concrete Paths fail if the object being targetted does not exist


The [Haskel path](https://hackage.haskell.org/package/path) package additionally (again) differenciates FilePaths from DirectoryPaths. This allows them to have rules that stop any path's being concatenated onto a FilePath, sine that is terminal

The question of how far down this path to go is an open one.
It might be something better resolved with traits than the normal hierachical type system.
A nice feature of defining `AbstractPaths`, is if desired different packages can define more (or less) rigerious definations of a filepath. And user's choosing to use such libraries will find they interact seemlessly with Base and with any other package.


# Pre-empted Questions:

## Why do you want to represent things other than file paths?


Because Julia is data-obsessed, but we shouldn’t prescribe where the data is located.
As a package writer we don’t care if the data is on a local disk, a network disk, hosted on a website, in a database, or even as a element of a XML document. So long as whereever it is hosted, it exposes a API meeting out needs. 
Basicaly so long as `open` works on the location description the end user provides; then it is the users problem to ensure it meets their other format requirements (Eg don't open a JSON file, with an XML Parser).

I was started on this, because I was using [WordNet.jl](https://github.com/jbn/WordNet.jl),
And I wanted to shift my data to a [Swift Object Store](http://docs.openstack.org/developer/swift/) -- which apparently all the cool kids working with DNA/Protein datasets have started doing (take that with a grain of salt, it came from our Open Stack sys-admin).

So I thought I would define in [SwiftObjectStores.jl](https://github.com/oxinabox/SwiftObjectStores.jl/) a `SwiftPath`, and it would overload `open`, and all would be well.


## Why should this be in Base? You could have it in a package.

This shoud be in Base, rather than in a package, because only Base can deprecate `open(::AbstractString)`.
It is important to deprecate `open(::AbstractString)` because this will force all packges to use `open(::AbstractPath)`.
Thus achieving the goal of the user being able to store there data where ever they want.
Otherwise package authors will probably never update since it would mean adding another dependancy, to even make it an option.

## Why not subtype AbstractString?
A Symbol is not a string because semantically it is a name, a Path is not a string because semantically it is an address.

Brett Cannon, who worked on Python3's PathLib, has written [a blog post about this](https://snarky.ca/why-pathlib-path-doesn-t-inherit-from-str/).
His key point is that while all path can be represented as strings, they are semantically very different, supporting different operations -- you can't just append a random string to a path, and get back a path. See some of the discussion on path operations in [my own blog post, linked earlier](http://white.ucc.asn.au/2016/09/14/an-algebraic-structure-for-path-schema-take2.html).


Further, to this point, with the highly abstracted defintion of a path, some useful paths don't have nice string representations.
Consider that a path contains all the information that is required to open a location.
For a networked store, this could include an authentication token, in the form of a binary blob.

Subtyping Abstract String does work (I think), and even works with packages without them having to change there code.
However, it is quiet the hack, and I think the result would be rather messy.

## Wait, you said a Glob was a kind of path, but glob's return multiple different files ?? :-S

Yes, Globs like XPATHs, are what I call [MultiPaths in my post](http://white.ucc.asn.au/2016/09/14/an-algebraic-structure-for-path-schema-take2.html).
They point to multiple locations.
We might want to handle them differently to normal  paths (MonoPaths).
Either by not considering them at all.
Or by adding a type "AbstractMultiPath".
I am dubious as to what it means to `open` them in a practical sense.
It would be nice to be able to require them to have a method, `resolve_to_monopaths`.
Which would return an iterator of monopath address -- one for each location they point at.
I am not sure on the practicality of this.

In most ways though, they act very similar to AbstractPaths.
