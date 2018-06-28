# Garbage Collector Extensions

- **Title:** Garbage collector extensions for better foreign language support
- **Author:** Reimer Behrends <behrends@gmail.com>
- **Created:** May 2018
- **Status:** work in progress

## Introduction

The support for modules written entirely or partly in foreign languages
to interface with the Julia GC or to use the GC for allocations that do
not neatly fit Julia's type system or use low-level approaches not
available in Julia (such as irregular data structure layouts) is
currently somewhat limited.

This proposal aims at fleshing out the API for allowing more complex
interaction of foreign code with the GC, especially the use of long-lived
foreign objects that are inextricably interwoven with Julia
objects.

Specific use cases that we are trying to address are:

* *Allowing the Julia GC to manage foreign objects with arbitrary
  layouts.* Not all objects -- especially those from preexisting
  libraries -- fit Julia's type system, for example, specialized
  container types written in C/C++. Such objects can comprise multiple
  memory blocks that require a custom marking mechanism and may also
  require finalizers written in C.
* *Providing additional roots to the GC.* Currently, to have additional
  roots, they must be stored in a location that is visible to Julia.
  This can be expensive if such roots are updated frequently or are
  contained in data structures that would have to be laboriously
  translated into a format usable by Julia. Instead, we want to allow
  for roots to be discoverable at the beginning of a garbage collection.
* *Conservative scanning of stack frames and objects.* Currently,
  scanning does have to be precise. If we desire to use the GC for
  foreign code that requires conservative scanning (especially for
  foreign stack frames), then it is necessary to have functionality
  that determines whether a machine word is a pointer to an object,
  including to its interior.

To demonstrate the applicability and viability of these mechanisms, we
have fully integrated Julia with the GAP computer algebra system, to the
point that GAP's regular garbage collector is completely replaced with
Julia's and that the lifetime of all GAP objects is entirely managed by
Julia. We also implemented a self-contained test program using these
mechanisms, and integrated it into the Julia test suite.

The proposed implementation should not incur measurable overhead for
Julia itself, as it only exposes additional functionality that is unused
by Julia code, plus functionality hooks that are designed to only incur
a few clock cycles of overhead per garbage collection. More specific
discussion of overhead can be found accompanying the descriptions of
these hooks.

An implementation of this proposal can be found on GitHub under
https://github.com/rbehrends/julia (branch `rb/gc-extensions`). See
the example in the `test/gcext` subdirectory for an example of using
this API.

An implementation of GAP that uses the Julia GC in lieu of its native
GC can likewise be found on GitHub at https://github.com/rbehrends/gap
(branch `alt-gc`). This version of GAP can be built with:

    ./autogen.sh
    ./configure --with-gc=julia --with-julia=/path/to/julia/usr
    make

## Callbacks

In order to allow foreign code to have access to necessary functionality
in the garbage collector, we allow foreign code to register callbacks for
certain GC events. We provide for six types of callbacks:

* `pre_gc` (beginning of garbage collection).
* `post_gc` (end of garbage collection).
* `root_scanner` (before scanning GC roots).
* `task_scanner` (when scanning Julia tasks).
* `notify_external_alloc` (when an external object is allocated).
* `notify_external_free` (when an external object is freed).

With external objects, we refer to what in the current implementation are
called `bigval_t` objects. These are allocated using the system's memory
allocator rather than using Julia's external allocator. In order to not
expose the implementation detail, we talk about "internal" and "external"
objects rather than objects that are allocated as part of Julia's object
pool or through system routines, respectively.

For each type of callback, there is a corresponding function pointer type.
Registering and deregistering callbacks occurs via macros.

```
typedef void (*jl_gc_cb_pre_gc_t)(int full);
typedef void (*jl_gc_cb_post_gc_t)(int full);
typedef void (*jl_gc_cb_root_scanner_t)(int full);
typedef void (*jl_gc_cb_task_scanner_t)(jl_task_t *task, int full);
typedef void (*jl_gc_cb_external_alloc_t)(void *addr, size_t size);
typedef void (*jl_gc_cb_external_free_t)(void *addr);

#define jl_gc_register_callback(cb, func)
#define jl_gc_deregister_callback(cb, func)
```

The `jl_gc_register_callback()` and `jl_gc_deregister_callback()` macros
are called with the appropriate index and a function pointer of the
matching type. Each function can be registered at most once; duplicate
registrations will be discarded.

*Performance impact:* The callback implementation is designed to incur
negligible overhead if no callbacks are used and no more overhead than
necessary to invoke the callbacks. The callbacks are all kept in linked
lists; if no callbacks are registered, all that is done is testing a
static variable for being null and to branch if it is. As branch
behavior should always be the same, only a few clock cycles are used, as
long as the variable is in the cache and the branch target in the BTB.

## Additional GC roots and hooking into the GC process

We provide three callbacks that are called at the beginning of a GC
(`pre_gc`), the beginning of the mark phase (`root_scanner`),
and the end of the GC (`post_gc`).

As these callbacks are tested and called only once per collection, overhead
should be negligible.

Additional roots can be marked from the root scanner callback by
calling the `jl_gc_mark_queue_obj()` function, which takes a pointer to
the current thread's thread-local storage a pointer to the object as its
parameters.

```
int jl_gc_mark_queue_obj(jl_ptls_t ptls, jl_value_t *obj);
```

The `ptls` parameter can be filled in from the return value of
the `jl_get_ptls_states()` function, which returns a pointer to
the thread-local storage of the current thread.

The return value of `jl_gc_mark_queue_obj()` can be ignored for marking
roots, but will be relevant for marking foreign objects (see below).

## Managing foreign objects with custom layouts

Foreign objects with custom layouts can define their own datatype through
the `jl_new_foreign_type()` function. Its first three parameters are the
same as for regular data types; following are a pointer to a mark function
and a pointer to a finalizer function (the latter of which can be null).

The mark function argument (`markfunc`) and the finalizer function
(`finalizefunc`) take function pointer arguments; the finalizer function
can be `NULL`.

The `haspointers` parameter should be non-zero if
the object may contain references to Julia objects; the `large` parameter
should be non-zero if the size is greater than the value returned by
`jl_gc_max_internal_obj_size()` and zero otherwise. If the object can be
both larger or not, then two distinct foreign types need to be created, one
for the case where the size is less than or equal and one for the case where
it is larger than the value of `jl_gc_max_internal_obj_size()`.

```
typedef uintptr_t (*jl_markfunc_t)(jl_ptls_t ptls, jl_value_t *obj);
typedef void (*jl_finalizefunc_t)(jl_value_t *obj);
jl_datatype_t *jl_new_foreign_type(
  jl_sym_t *name,
  jl_module_t *module,
  jl_datatype_t *super,
  jl_markfunc_t markfunc,
  jl_finalizefunc_t finalizefunc,
  int haspointers,
  int large
);
size_t jl_gc_max_internal_obj_size(void);
```

*Performance impact:* Custom mark functions need to be called during the
performance-critical mark loop of the garbage collector. In order to
avoid overhead for the other cases, the code is engineered to consider
such objects as the last possible option in the existing if-else chains.
To accomplish that, such foreign types use the existing
`jl_datatype_layout_t` structure, with `fielddesc_type` set to `3`,
which is looked at after the other data types and the other alternatives
for `fielddesc_type`.

The mark function gets passed a pointer to thread-local storage (`ptls`)
and the object to be marked (which will be of the type defined through
`jl_new_foreign_type()`. The `ptls` argument is an optimization so that
`jl_get_ptls_states()` does not need to be called unnecessarily during
the mark loop.

The mark function implementation also uses `jl_gc_mark_queue_obj()` to
mark objects, as with the root scanner; however, in contrast to marking
roots, the return value cannot be ignored. Per object, the mark function
should count how often `jl_gc_mark_queue_obj()` for subjects return
non-zero values and return that number. If an object has no subobjects,
the mark function should return zero.

This information is relevant for the generational part of garbage
collection. The return value of `jl_gc_mark_queue_obj()` is non-zero
if a young generation object has been marked. When the mark function
has been called for an old object and the mark function returns a
non-zero value (thus showing how many young objects have been marked
from the old one one), the GC knows to update its internal data
structures accordingly.

For an example of this, see the `gcext` test in the Julia repository,
which defines a couple of such custom mark functions.

To enable C finalization for such an object, the function
`jl_gc_set_needs_foreign_finalizer()` has to be called for an
object. This object has to be of a foreign type and that foreign
function has to be defined with a non-null finalizer function.

```
JL_DLLEXPORT void jl_gc_set_needs_foreign_finalizer(jl_value_t *obj);
```

On the C side, such objects can be allocated using the call
`jl_gc_alloc_typed()`; the function takes a pointer to the thread's
thread-local storage, the desired size, and the foreign datatype as its
arguments.

```
JL_DLLEXPORT void * jl_gc_alloc_typed(jl_ptls_t ptls, size_t sz,
  void *ty);
```

## Conservative scanning

Some external modules may require conservative scanning, especially
of the stack. This was the case for example, with our application
involving the GAP language.

In order to handle conservative scanning, we need to expose the fact
that Julia distinguishes between objects it manages itself (which we
call "internal objects" in this document) and objects that it manages
via "malloc()" or similar calls (these we call "external objects").

The proposed functionality relies on calls to Julia to determine
if a pointer is a reference to an internal object, but leaves it up
to the author of the foreign code to determine this for external
objects; to this end, we provide callbacks to notify foreign code
of the allocation (`notify_external_alloc`) or deallocation
(`notify_external_free`).

The accompanying function pointer types are:

```
typedef void (*jl_gc_cb_external_alloc_t)(void *addr, size_t size);
typedef void (*jl_gc_cb_external_free_t)(void *addr);
```

The allocation callback is invoked with the address and size of the
new object, the deallocation callback is invoked with the address of
the object about to be freed. Allocation and deallocation is still
managed by Julia. The intent here is that foreign code can track
allocations and deallocations in a data structure of its own if
needed. An example of this can be seen in the `gcext` test, where
we use a balanced tree to track allocations.

*Performance impact:* The overhead for the callbacks should be minimal,
especially since the cost of allocating large objects through the system
allocator and initializing them will dominate the allocation process.

Note that some of these objects may not have a valid type field and
especially in the context of conservative scanning, pointers to
objects with invalid type fields may inadvertently be generated. In
such a case, the validity of the type field should also be checked,
e.g. with: `jl_gc_internal_obj_base_ptr(jl_typeof(obj)) != NULL` (see
below for the semantics of this function).

To determine whether a pointer points to an internal object, programmers
can use the following functions:

```
jl_value_t *jl_gc_internal_obj_base_ptr(void *p);
int jl_gc_is_internal_obj_alloc(jl_value_t *p);
```

The `jl_gc_internal_obj_base_ptr()` function returns `NULL` if the
argument does not point to the beginning, the interior, or the end of an
internal object. otherwise, it returns a pointer to the beginning of the
object it points to. The `jl_gc_is_internal_obj_alloc()` function is an
optimized fast path version; it returns a non-zero value if and only the
argument is a valid internal object or if it points to memory reserved
for the allocation of such objects. In the latter case, it is guaranteed
that the type field of such an object does not contain a valid datatype.

In addition, we also provide a hook to conservatively scan task stacks
(`jl_gc_cb_task_scanner`):

```
typedef void (*jl_gc_cb_task_scanner_t)(jl_task_t *task, int full);
```

This task scanner hook functions like the root scanner hook, except
that it is called for each task and with a pointer to the task object
as an argument.

## Performance evaluation

In order to evaluate the changes for performance, we ran the system
with no callbacks or foreign types installed against the Julia base
benchmarks (namely, the "array", "collection", "micro", "shootout",
"sparse", "string", and "tuple" suites) for both our changes and
a recent version of the master branch.

We did not observe any performance regressions in those benchmarks.
While, due to the noisiness of our test system, spurious regressions
crept up occasionally (about a handful per run), none of them
persisted for more than one run of the suite.

Also, when testing for improvements, similar and similarly common
performance changes occurred in the other direction, including spurious
"regressions" of the master branch compared to our changed version.

Finally, we ran a couple of specialized microbenchmarks (included below)
designed to stress-test the garbage collector several times and observed
the performance over several runs for both the master branch version and
our changes; we did not observe significant differences in the
distribution of `@btime` results.

    using BenchmarkTools

    function bencharr(n, m, m2, x, y)
      global t = [ (x, y) for i in 1:(n * n * m2) ]
      local a = [ [ (x, y) for j in 1:n ] for i in 1:n ]
      for i in 1:m
        a = map(outer -> map(inner -> (inner[2], inner[1]), outer), a)
      end
    end

    function fac(n)
      local result = BigInt(1)
      for i in 2:n
        result *= i
      end
      return result
    end

    function benchfac(n)
      local total = BigInt(0)
      for i in 1:n
        total += fac(i)
      end
      return total
    end

    print("Array allocations benchmark:  ")
    @btime bencharr(200, 1000, 100, "x", "y")
    print("BigInt allocations benchmark: ")
    @btime benchfac(2000)

