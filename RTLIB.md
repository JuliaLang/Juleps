# JULEP RTLIB

- **Title:** A runtime library for Julia
- **Authors:** Valentin Churavy <<v.churavy@gmail.com>>
- **Created:** November 11, 2016
- **Status:** work in progress

## Introduction

Currently there are two implementation of intrinsics supported in Julia. One of the
implementations is defined in `runtime_intrinsics.c` and the second one is defined in
`intrinsics.cpp` on top of [LLVM intrinsics](http://llvm.org/docs/LangRef.html#id1190)
and [LLVM instructions](http://llvm.org/docs/LangRef.html#instruction-reference).

The first implementation specifies the semantics and behavior of the intrinsics and is
used as a fallback. The LLVM based implementation is best understood as a pure performance
optimization.

When a LLVM intrinsic is used and the compiler can't generate hardware instructions
for it, a library call to the runtime library (compiler-rt or libgcc) is emitted.
As a result Julia code using LLVM as a codegen backend needs to link against a
runtime library. In the exploratory work (see https://github.com/JuliaLang/julia/pull/18734)
the LLVM compiler-rt library was choosen, due to its MIT license.

A drawback of compiler-rt is that it is not fully portable. As an example
`Float128` support is missing on 32bit platforms.

## Motivating problem

How do we support `Float16` and `Float128` in a portable and performant manner?
The current implementation of `Float16` support in Julia is eagerly resolving to
promotion to `Float32` in order to implement most operations. This precludes optimization
on platforms that natively support `Float16` (most prominently GPUs). The second
problem is how are we going to support `Float128` across all platforms in a portable
and still performant way. On 32bit systems we cannot rely on platform implementations.

## Goals

Implement a runtime-library in a mix of C and Julia that contains Julia intrinsics
and compiler-rt. This would allow us to use optimized implementations from compiler-rt,
while having the flexibility of Julia to implement non-essential intrinsics.

When Julia uses LLVM as a compiler backend it should use a lazy libcall scheme.
When the compiler can't emit optimized code paths (e.g. LLVM instructions and intrinsics),
it will emit calls to calls to the intrinsics. Currently this done already for
better error reporting. The interpreter can eagerly resolve to the implementations
in the runtime library, which will require `ccall`for the `C` based part of the
runtime library.

The runtime library will consist of two stages. Stage-1 is implemented in C and
contains compiler-rt, while Stage-2 is implemented in Julia. Stage-1 is required
for bootstrapping a minimal Julia, on which Stage-2 can be implemented.
Stage- 1 will also contain optimized (and platform dependent) implementations of
the intrinsics, while Stage-2 will contain portable and general implementations.

Another goal is to define what is the minimal set of intrinsics that Julia
requires (Stage-1) and what is the extended set (Stage-2). A well defined set of
intrinsics would also be beneficial for alternative compilers.

### Stage 1

C-based implementation of the essential Julia intrinsics + compiler-rt.
- `jl_reinterpret`
- `jl_pointerset`
- `jl_pointerref`
- Operations on Integers
  - Arithmetic
  - Comparisons
  - Conversion between integers
- Operations on Floating Point (hardware based)
  - Arithmetic
  - Comparisons
  - Conversion to and from integers
  - `Float32` and `Float64` support

### Stage 2

Julia based implementation for non-essential Julia intrinsics and implementations
to supplement compiler-rt. This implementation will based on a reduced base library
(early stages of the sysimage) that will is only allowed to use Stage-1 funcionality.
The proper sysimage will be based upon Stage-1 and Stage-2.
The basic idea is https://github.com/JuliaLang/julia/pull/18927,
which contains the initial port of compiler-rt to Julia.
- Operations on Floating Point (software based)
  - Arithmetic
  - Comparisons
  - Conversion to and from integers
  - Necessary for `Float16` and `Float128` support
- Scalar implementations for vectorized instructions

#### Building Stage-2
1. Build Stage-1 and create a shared object file `rtlib-stage1.so`
2. Build inference.ji
3. Build Stage-2 with `--rtlib=rtlib-stage1.so` and `--sysimage inference.ji`
4. Take the object files from Stage-1 and Stage-2 and create `rtlib.so` containing
   both Stage-1 and Stage-2.
5. Build `sys.so` with `--rtlib=rtlib.so` and `--sysimage inference.ji`

## Testing and Benchmarking

All implementations, but especially the runtime versions should be thoroughly
tested for correctness and performance. As part of this Julep the testsuite needs
to be extended to cover the current and future runtime intrinsics.

## Non-Goals

- No support for atomics at this time. Julia will continue to use `libatomic`.
