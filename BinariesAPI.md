# JULEP BinariesAPI

- **Title:** A structured HTTP API for getting Julia binaries
- **Authors:** Eric Davies <<iamed2@gmail.com>>
- **Created:** June 16, 2017
- **Status:** proposed

## Abstract

The URLs for fetching Julia binaries of various versions have changed a few times across
releases.
The latest change necessitated PRs to multiple repos to change AppVeyor scripts.
This Julep proposes a level of abstraction over the Julia binary URLs which is decoupled
from the storage service used for the binaries.

## Current Status

The current (as of this Julep's creation) [posted URLs](https://julialang.org/downloads)
for Julia binaries are (using 64-bit Linux and 32-bit Windows for example):

- 0.5 (release)
  - https://julialang-s3.julialang.org/bin/linux/x64/0.5/julia-0.5.2-linux-x86_64.tar.gz
  - https://julialang-s3.julialang.org/bin/winnt/x86/0.5/julia-0.5.2-win32.exe
- 0.6 (release candidate)
  - https://julialang-s3.julialang.org/bin/linux/x64/0.6/julia-0.6-latest-linux-x86_64.tar.gz
  - https://julialang-s3.julialang.org/bin/winnt/x86/0.6/julia-0.6-latest-win32.exe
- 0.7 (nightly)
  - https://status.julialang.org/download/linux-x86_64
  - https://status.julialang.org/download/win32

Recently, PRs were made to update binary URLs for AppVeyor scripts.
The URL for the nightly build was different than the above:

- https://julialangnightlies-s3.julialang.org/bin/winnt/x86/julia-latest-win32.exe

Source code links are also provided, linking to GitHub's auto-generated tarballs.

## General Proposal

Set up a domain (e.g., binaries.julialang.org) for hosting binaries, with the URLs isolated
from the service hosting the files (e.g., Amazon API Gateway over S3).
Introduce a standard format for the Julia binary URLs, respected by every release.

## URL Format Proposal

`https://binaries.julialang.org/<os>/<arch>/<version>`

### OS

`<os>` is one of `linux`, `windows`, `macos`, and any others which may be added in the
future (e.g. `freebsd`).

### Arch

`<arch>` is one of `x86`, `x86_64`, `arm`, `ppc64le`, and any others which may be added in
the future (e.g. `aarch64`).
These are the abbreviations used in the
[LLVM source](http://llvm.org/docs/doxygen/html/Triple_8h_source.html).

### Version

`<version>` is a Julia version prefix.
Any trailing unspecified components are assumed to be the maximum available value once
all other conditions are satisfied.

#### Valid Version Values

All values must be `release` or `nightly` or match this specification:

```regex
^v\d+(\.\d+)?(\.\d+)?(-\w+)?([\+\.]\w+)?$
```

Assuming the latest nightly is `v0.7.0-DEV.568`, the latest release is `v0.5.2`, and
the latest 0.6 release candidate is `v0.6.0-rc3`:

- `v0`, `v0.7`, `v0.7.0`, `v0.7.0-DEV`, and `v0.7.0-DEV.568` would match `v0.7.0-DEV.568`
- `v0.6`, `v0.6.0`, and `v0.6.0-rc3` would match `v0.6.0-rc3`
- `v0.5` and `v0.5.2` would match `v0.5.2`
- `v0.5.0` would match `v0.5.0`
- `release` would match `v0.5.2`
- `nightly` would match `v0.7.0-DEV.568`

The following versions match the regex but would not resolve to an existing version:

- `v0.8`
- `v0.6.0-rc4`
- `v0.7.0-DEV.100000`
- `v0.5.3`
- `v0.6.0-cromulent`
- `v1`

### Possible Extensions

URL parameters (`?key1=value1&key2=value2`) could be added to the URL in case several
builds are created for a single version with different features enabled.
For example, if there were threaded and non-threaded builds, `threaded={0,1}` could be
a parameter selection, e.g., `https://binaries.julialang.org/macos/x86_64/release?threaded=1`

### Failures



## Deprecation strategy

## Issues Beyond the Scope of This Julep

