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

1. *Allowing the Julia GC to manage foreign objects with arbitrary
  layouts.* Not all objects -- especially those from preexisting
  libraries -- fit Julia's type system, for example, specialized
  container types written in C/C++. Such objects can comprise multiple
  memory blocks that require a custom marking mechanism and may also
  require low-level finalizer behavior written in C.
2. *Providing additional roots to the GC.* Currently, to have additional
  roots, they must be stored in a location that is visible to Julia.
  This can be expensive if such roots are updated frequently or are
  contained in data structures that would have to be laboriously
  translated into a format usable by Julia. Instead, we want to allow
  for roots to be discoverable at the beginning of a garbage collection.
3. *Conservative scanning of stack frames and objects.* Currently,
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
<https://github.com/rbehrends/julia> (branch `rb/gc-extensions`). See
the example in the `test/gcext` subdirectory for an example of using
this API.

An implementation of GAP that uses the Julia GC in lieu of its native
GC can likewise been found on GitHub at <https://github.com/rbehrends/gap>
(branch `alt-gc`). That version of GAP can be built with:

    ./autogen.sh
    ./configure --with-gc=julia --with-julia=/path/to/julia/usr
    make

## Callbacks

In order to allow foreign code to have access to necessary functionality
in the garbage collector, we allow foreign code to register callbacks for
certain GC events. We provide for six types of callbacks:

1. Beginning of garbage collection (`pre_gc`)
2. End of garbage collection (`post_gc`)
3. When scanning GC roots (`root_scanner`)
4. When scanning Julia tasks (`task_scanner`)
5. When an external object is allocated (`notify_external_alloc`).
6. When an external object is deallocated (`notify_external_free`).

These callbacks are *not* per se thread-safe. It is up to to the callback
implementation to ensure that no violations of thread-safety occur.

In particular, each of these can be called from any thread. All except the
first two can be called concurrently. In the current Julia GC implementation,
the `post_gc` callback may also not be called before the next `pre_gc`.

With external objects, we refer to what in the current Julia implementation are
called `bigval_t` objects. These are allocated using the system's memory
allocator rather than using Julia's external allocator. In order to not
expose this implementation detail, we talk about "internal" and "external"
objects rather than objects that are allocated as part of Julia's object
pool or through system routines, respectively.

For each type of callback, there is a corresponding function pointer type.
Registering and deregistering callbacks occurs via corresponding setter
functions.

```
typedef void (*jl_gc_cb_pre_gc_t)(int full);
typedef void (*jl_gc_cb_post_gc_t)(int full);
typedef void (*jl_gc_cb_root_scanner_t)(int full);
typedef void (*jl_gc_cb_task_scanner_t)(jl_task_t *task, int full);
typedef void (*jl_gc_cb_notify_external_alloc_t)(void *addr, size_t size);
typedef void (*jl_gc_cb_notify_external_free_t)(void *addr);

void jl_gc_set_cb_root_scanner(jl_gc_cb_root_scanner_t cb, int enable);
void jl_gc_set_cb_task_scanner(jl_gc_cb_task_scanner_t cb, int enable);
void jl_gc_set_cb_pre_gc(jl_gc_cb_pre_gc_t cb, int enable);
void jl_gc_set_cb_post_gc(jl_gc_cb_post_gc_t cb, int enable);
void jl_gc_set_cb_notify_external_alloc(jl_gc_cb_notify_external_alloc_t cb, int enable);
void jl_gc_set_cb_notify_external_free(jl_gc_cb_notify_external_free_t cb, int enable);
```

For each setter function, a callback function is supplied, along with a flag
(`1` for enabling the callback, `0` for removing it again). Attempting to
register a callback multiple times will only register it once.

*Performance impact:* The callback implementation is designed to incur
negligible overhead if no callbacks are used and no more overhead than
necessary to invoke the callbacks. The callbacks are all kept in linked
lists; if no callbacks are registered, all that is done is testing a
static variable for being null and to branch if it is. As branch
behavior should always be the same, only a few clock cycles are used, as
long as the variable is in the cache and the branch target in the BTB.

## Additional GC roots and hooking into the GC process

We provide three callbacks that are called at the beginning of a GC
(`pre_gc`), the beginning of the mark phase (`root_scanner`), and the end of
the GC (`post_gc`). As these callbacks are tested and called only once per
collection, overhead should be negligible. The `full` argument passed
to these callbacks indicates whether this is a full or partial garbage
collection.

In addition, we also provide a `task_scanner` hook, which functions like
the `root_scanner` hook, except that it is called for each task and with
a pointer to the task object as its first argument.

Additional roots can be marked from the `root_scanner` and
`task_scanner` callbacks by calling the `jl_gc_mark_queue_obj()`
function, which takes a pointer to the current thread's thread-local
storage a pointer to the object as its parameters.

```
int jl_gc_mark_queue_obj(jl_ptls_t ptls, jl_value_t *obj);
```

The `ptls` parameter can be filled in from the return value of
the `jl_get_ptls_states()` function, which returns a pointer to
the thread-local storage of the current thread.

The return value of `jl_gc_mark_queue_obj()` can be ignored for marking
roots, but will be relevant for marking foreign objects (see below).

When processing large objects, calling `jl_gc_mark_queue_obj()` can be
ineffecient, as each object will be pushed on the mark stack separately.

If possible, it is therefore recommended that programmers use the
following function, designed for arrays of references, which handles
this use case more efficiently:

```
void jl_gc_mark_queue_objarray(jl_ptls_t ptls, jl_value_t *parent,
    jl_value_t **objs, size_t nobjs);
```

Here, `parent` is a reference to the current object, `objs` is a pointer
to the start of an array of object references, and `nobjs` is the number
of object references contained in that array. That array must be part of
the object; it must not be allocated in static memory or on the stack.

Unlike `jl_gc_mark_queue_obj()`, this function does not have a return
value, as it does the requisite tracking itself.

Calling this function will only require one slot on the mark stack, as
opposed to the `nobjs` slot that individual calls to
`jl_gc_mark_queue_obj()` would require, making it considerably more
memory efficient.

## Managing foreign objects with custom layouts

Foreign objects with custom layouts can define their own datatype through
the `jl_new_foreign_type()` function:

```
typedef uintptr_t (*jl_markfunc_t)(jl_ptls_t ptls, jl_value_t *obj);
typedef void (*jl_sweepfunc_t)(jl_value_t *obj);

jl_datatype_t *jl_new_foreign_type(
  jl_sym_t *name,
  jl_module_t *module,
  jl_datatype_t *super,
  jl_markfunc_t markfunc,
  jl_sweepfunc_t sweepfunc,
  int haspointers,
  int large
);
```

The first three parameters of `jl_new_foreign_type` are the same as for
regular data types; following are a pointer to a mark function
(`markfunc`) and a pointer to a sweep function (`sweepfunc`); the latter
of which can be null.

The `haspointers` parameter should be non-zero if instances of the new
datatype may contain references to Julia objects; the `large` parameter
should be non-zero if the size of instances of the new datatype will be
greater than the value returned by `jl_gc_max_internal_obj_size()` and
zero otherwise. If the objects can be both larger or not, then two
distinct foreign types need to be created, one for the case where the
size is less than or equal and one for the case where it is larger than
the value of `jl_gc_max_internal_obj_size()`.

```
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

### Mark functions for foreign objects

The mark function `markfunc` gets passed a pointer to thread-local
storage (`ptls`)
and the object to be marked (which will be of the type defined through
`jl_new_foreign_type()`. The `ptls` argument is an optimization so that
`jl_get_ptls_states()` does not need to be called unnecessarily during
the mark loop.

The mark function implementation also uses `jl_gc_mark_queue_obj()` to
mark objects, as with the `root_scanner` callback; however, in contrast to marking
roots, the return value cannot be ignored. Per object, the mark function
should count how often `jl_gc_mark_queue_obj()` for subjects return
non-zero values and return that number. If an object has no subobjects,
the mark function should return zero.

This information is relevant for the generational part of garbage
collection. The return value of `jl_gc_mark_queue_obj()` is non-zero
if a young generation object has been marked. When the mark function
has been called for an old object and the mark function returns a
non-zero value (thus showing how many young objects have been marked
from the old one), the GC knows to update its internal data
structures accordingly.

For an example of this, see the `gcext` test in the Julia repository,
which defines a couple of such custom mark functions.

### Sweep functions for foreign objects

Sweep functions for foreign objects are similar to, but more limited
than finalizers, as they are not intended to replace finalizer
functionality. Rather, they are meant to clean up complex memory
structures allocated with raw malloc calls or operating system
resources. They will be called during the sweep phase and must not have
side effects that are visible to Julia.

To enable sweep functions for a foreign object, the function
`jl_gc_schedule_foreign_sweepfunc()` has to be called on the object,
which has to be of a foreign type and that foreign function has to be
defined with a non-null sweep function `sweepfunc`. Without that call,
the sweep function will not be called on this particular object. This is
to avoid unnecessary overhead if not all objects of that type require
extra sweep phase semantics. This function should be called at most once
per object; if called multiple times, the sweep function may be invoked
more than once on the given object.

```
JL_DLLEXPORT void jl_gc_schedule_foreign_sweepfunc(jl_ptls_t ptls,
        jl_value_t *obj);
```

### Allocating foreign objects

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
of the stack. This was the case, for example, with our application
involving the GAP computer algebra system.

We note that conservative scanning should be avoided if at all possible;
it is not intended as a way to avoid tracking Julia references (for which
the `root_scanner` callback and custom marking functions offer efficient
options if other approaches fail), but as a feature of last resort if
integrating an existing codebase through other means is not viable.

Conservative scanning must be enabled through a call to the following
function:

```
void jl_gc_enable_conservative_scanning(void);
```

This function can be called from C code both before and after `jl_init()` 
and is thread-safe. Enabling this introduces a very small, but non-zero
overhead, which is why it is not enabled by default.

In order to handle conservative scanning, we need to expose the fact
that Julia distinguishes between objects it manages itself (which we
call "internal objects" in this document) and objects that it manages
via "malloc()" or similar calls (these we call "external objects").

The proposed functionality relies on calls to Julia to determine
if a pointer is a reference to an internal object, but leaves it up
to the author of the foreign code to determine this for external
objects; to this end, we provide callbacks to notify foreign code
of the allocation or deallocation of such objects.

The accompanying function pointer types are:

```
typedef void (*jl_gc_cb_notify_external_alloc_t)(void *addr, size_t size);
typedef void (*jl_gc_cb_notify_external_free_t)(void *addr);
```

The allocation callback is invoked with the address and size of the
new object, the deallocation callback is invoked with the address of
the object about to be freed. Allocation and deallocation is still
managed by Julia. The intent here is that foreign code can track
allocations and deallocations in a data structure of its own if
needed. An example of this can be seen in the `gcext` test, where
we use a balanced tree to track allocations.

Note that registering such callbacks will only track allocations that
occur *after* the callbacks have been set. We assume here that the client
is only interested in tracking its own objects that may be stored in
opaque stack frames, but not other Julia objects that may be passed in
from Julia calls. If the client needs to track *all* allocations, then
the callbacks *must* be registered before calling `jl_init()`.

*Performance impact:* The overhead for the callbacks should be minimal,
especially since the cost of allocating large objects through the system
allocator and initializing them will dominate the allocation process.

Note that some of these objects may not have a valid type field and
especially in the context of conservative scanning, pointers to
objects with invalid type fields may inadvertently be generated. In
such a case, the validity of the type field should also be checked,
e.g. with: `jl_gc_internal_obj_base_ptr(jl_typeof(obj)) != NULL` (see
below for the semantics of this function).

To determine whether a pointer points to an internal object, the
following functions may be used:

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

