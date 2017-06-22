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
* Zero logger setup for simple uses.
* Easy filtering of log messages.
* Clear guidelines about the meaning and appropriate use of standard log levels
  for consistency between packages, along with guiding the appropriate use of
  logging vs stdout.
* To encourage the use of logging over ad hoc console output, the default
  console log handler should emphasize the *log message*, and metadata should be
  communicated in a non-intrusive way.


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

A rough work in progress implementation is available at
https://github.com/c42f/MicroLogging.jl


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


## Survey of other logging systems

Much of the above is a reinterpretation in julia of python's standard logging
module, attempting to be even more convenient and much more efficient, while
retaining the same flexibility.

* A summary of how this all fits in with Logging.jl, MiniLogging.jl, Memento.jl,
  LumberJack.jl, and any other logging libraries which can be found.
* glib (C) - https://developer.gnome.org/glib/stable/glib-Message-Logging.html
* a-cl-logger (Common lisp) - https://github.com/AccelerationNet/a-cl-logger
* Lager (Erlang) - https://github.com/erlang-lager/lager
* Some java logger *cough*?  That should be enterprisey enough, right ;-)
* syslog (unix) ?

TODO - much more here.

