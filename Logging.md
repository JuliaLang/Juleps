# JULEP Logging

- **Title:** A unified logging interface
- **Author:** Chris Foster <chris42f@gmail.com>
- **Created:** February 2017
- **Status:** work in progress

## Abstract

*Logging* is a tool for understanding program execution by recording the order and
timing of a sequence of events.  A *logging library* provides tools to define
these events in the source code and capture the event stream when the program runs.
The information captured from each event makes its way through the system as a
*log record*. The ideal logging library should give developers and users insight
into the running of their software by provide tools to filter, save and
visualize these records.

Julia has included simple logging in `Base` since version 0.1, but the tools to
generate and capture events are still immature as of version 0.6. For example,
log messages are unstructured, there's no systematic capture of log metadata, no
debug logging, inflexible dispatch and filtering, and the role of the code at
the log site isn't completely clear.  Because of this, Julia 0.6 packages use
any of several incompatible logging libraries, and there's no systematic way to
generate and capture log messages.

This julep aims to improve the situation by proposing:

* A simple, unified interface to generate log events in `Base`
* Conventions for the structure and semantics of the resulting log records
* A minimum of dispatch machinery to capture, route and filter log records
* A default backend for displaying, filtering and interacting with the log
  stream which makes the log record structure visible.

A non-goal is to create a complete set of logging backends - these can be
supplied by packages.

## Desirable features

The desirable features fit into three rough categories - simplicity, flexibility
and efficiency.

Logging should be **simple to use** so that package authors can reach for `info`
rather than `println()`.  We should have:

* A minimum of syntax - ideally just a logger verb and the message in most
  cases.  Context information for log messages (file name, line number, module,
  stack trace, etc.) should be automatically gathered without a syntax burden.
* Freedom in formatting the log message - simple string interpolation,
  `@sprintf` and `fmt()`, etc should all be fine.
* No mention of log dispatch should be necessary at the message creation site.
* Easy filtering of log messages.
* Clear guidelines about the meaning and appropriate use of standard log levels
  for consistency between packages, along with guiding the appropriate use of
  logging vs stdout.
* The default console log handler should integrate somehow with the display
  system to show log records in a way which is highly readable. Ideally logging
  should be a tool for package authors just as much as it is a tool for
  understanding production systems.


The API should be **flexible enough** for advanced users:

* For all packages using the standard logging API, it should be simple to
  intercept, filter and redirect logs in a unified and centrally controlled way.
* Log records are more than a string: loggers typically gather context
  information both lexically (eg, module, file name, line number) and
  dynamically (eg, time, stack trace, thread id).  The API should preserve this
  structured information.
* Users should be able to add structured information to log records, to be
  preserved along with data extracted from the logging context. For example, a
  list of `key=value` pairs offers a decent combination of simplicity and power.
* Formatting and dispatch of log records should be in the hands of the user if
  they need it. For example, a log handler library may need to write json
  records across the network to a log server.
* It should be possible to log to a user defined log context; automatically
  choosing a context for zero setup logging may not suit all cases.  For
  example, in some cases we may want to use a log context explicitly attached to
  a user-defined data structure.
* It should be possible to control log filtering per thread or task.
* Possible extensions
    * Unique message IDs based on code location for finer grained message
      filtering?

The design should allow for an **efficient implementation**, to encourage
the availability of logging in production systems; logs you don't see should be
almost free, and logs you do see should be cheap to produce. The runtime cost
comes in three flavours:

* Cost in the logging library, to determine whether to filter a message.
* Cost in user code, to construct quantities which will only be used in the
  log message.
* Cost in the logging library in collecting context information and
  to dispatch and format log records.


## Proposed design

A prototype implementation is available at https://github.com/c42f/MicroLogging.jl
### Quickstart Example

#### Frontend
```julia
# using Base.Log

# Logging macros
@debug "A message for debugging (filtered out by default)"
@info "Information about normal program operation"
@warn "A potentially problem was detected"
@error "Something definitely went wrong, but we recovered enough to continue"
@logmsg Log.Info "Explicitly defined info log level"

# Free form message formatting
x = 10.50
@info "$x"
@info @sprintf("%.3f", x)
@info begin
    A = ones(4,4)
    "sum(A) = $(sum(A))"
end

# Progress reporting
for i=1:10
    @info "Some algorithm" progress=i/10
end

# User defined key value pairs
foo_val = 10.0
@info "test" foo=foo_val bar=42
```


### What is a log record?

Logging statements are used to understand algorithm flow - the order and timing
in which logging events happen - and the program state at each event.  Each
logging event is preserved in a **log record**.  The information in a record
needs to be gathered efficiently, but should be rich enough to give insight into
program execution.

A log record includes information explicitly given at the call site, and any
relevant metadata which can be harvested from the lexical and dynamic
environment.  Most logging libraries allow for two key pieces of information
to be supplied explicitly:

* The **log message** - a user-defined string containing key pieces of program
  state, chosen by the developer.
* The **log level** - a category for the message, usually ordered from verbose
  to severe.  The log level is generally used as an initial filter to remove
  verbose messages.

Some logging libraries (for example
[glib](https://developer.gnome.org/glib/stable/glib-Message-Logging.html)
structured logging) allow users to supply extra log record information in the
form of key value pairs.  Others like
[log4j2](https://logging.apache.org/log4j/2.x/manual/messages.html) require extra information to be
explicitly wrapped in a log record type.  In julia, supporting key value pairs
in logging statements gives a good mixture of usability and flexibility:
Information can be communicated to the logging backend as simple keyword
function arguments, and the keywords provide syntactic hints for early filtering
in the logging macro frontend.

In addition to the explicitly provided information, some useful metadata can be
automatically extracted and stored with each log record.  Some of this is
extracted from the lexical environment or generated by the logging frontend
macro, including code location (module, file, line number) and a unique message
identifier.  The rest is dynamic state which can generally on demand by the
backend, including system time, stack trace, current task id.

### The logging frontend

### Dispatching records

### Early filtering


## Concrete use cases

### Base

In Base, there are three somewhat disparate mechanisms for controlling logging.
An improved logging interface should unify these in a way which is convenient
both in the code and for user control.

* The 0.6 logging system's `logging()` function with redirection based on module
  and function.
* The `DEBUG_LOADING` mechanism in loading.jl and `JULIA_DEBUG_LOADING`
  environment variable.
* The depwarn system, and `--depwarn` command line flag


## Inspriation

This Julep draws inspiration from many previous logging frameworks, and helpful
discussions with many people online and at JuliaCon 2017.

The Java logging framework [log4j2](https://logging.apache.org/log4j/2.x/) was a
great source of use cases, as it contains the lessons from at least twenty years
of large production systems.  While containing a fairly large amount of
complexity, the design is generally very well motivated in the documentation,
giving a rich set of use cases.  The julia logging libraries - Base in julia 0.6,
Logging.jl, MiniLogging.jl, LumberJack.jl, and particularly
[Memento.jl](https://github.com/invenia/Memento.jl) - provided helpful
context for the needs of the julia community.

Structured logging as available in
[glib](https://developer.gnome.org/glib/stable/glib-Message-Logging.html)
and [RFC5424](https://datatracker.ietf.org/doc/rfc5424/?include_text=1) (The
Syslog protocol) provide context for the usefulness of log records as key value
pairs.

For the most part, existing julia libraries seem to follow the design tradition
of the standard [python logging library](https://docs.python.org/3/library/logging.html),
which has a lineage further described in [PEP-282](https://www.python.org/dev/peps/pep-0282/).
The python logging system provided a starting point for this Julep, though the
design eventually diverged from the typical hierarchical setup.

TODO: Re-survey the following?
* a-cl-logger (Common lisp) - https://github.com/AccelerationNet/a-cl-logger
* Lager (Erlang) - https://github.com/erlang-lager/lager


