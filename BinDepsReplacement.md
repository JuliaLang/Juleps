# JULEP BinDepsReplacement

- **Title:** A BinDeps Replacement
- **Authors:** Tony Kelman <<@tkelman>>
- **Created:** August 16, 2017
- **Status:** work in progress

## Abstract

We would like to make it simple for package developers, and reliable and
seamless for package users across all commonly-used platforms, to have binary
dependencies in Julia packages. This Julep is primarily concerned with Julia packages that wrap C, C++, or Fortran shared libraries via ccall, but similar
concerns apply for command line programs interfaced via shelling out, and any
other compiled language that can export C-compatible interfaces. Dependencies
on code written in other languages by way of interfacing to those languages'
interpreters, as in PyCall or RCall, are out of scope for the initial version
of this Julep.

## Problems with BinDeps.jl

The most commonly used tool for Julia packages to provide binary dependencies
today is [BinDeps.jl](https://github.com/JuliaLang/BinDeps.jl). While it
sometimes does the job, it has frequently been a cause of installation
problems for users and frustration for package developers to get working
and keep that way.

1. Works differently on each platform

BinDeps allows multiple providers for a given library dependency. It can
interface with several platform-specific package managers, or download
precompiled binaries, or execute source build instructions. The various
options mean there are many possible combinations that need to be tested
and supported.

2. Using system wide package managers requires root access

For users on shared Linux servers, e.g. departmental clusters or some corporate
environments or services like JuliaBox, installing packages via the system
wide package manager is not an allowed option. This is a major limitation of
BinDeps' strategy of relying on other package managers.

3. Different names and versions of libraries in each external package manager

BinDeps doesn't try to be a package manager, but it does try to interface with
several: apt-get on Debian and Ubuntu, yum on CentOS and Fedora, pacman on
Arch, zypper on OpenSUSE, FreeBSD pkg, Homebrew on MacOS, cross-compiled RPMs
via WinRPM.jl on Windows, etc. Unfortunately each of these package managers
has different naming conventions for the same libraries. Different package
repositories have different updating policies, so the available versions can
vary widely. If the distribution version of a library is too old to have a
feature or bug fix that the Julia wrapper package relies on, then users on
that distribution often have no choice but to compile the library from source.
Since the external package repositories are not under control of the Julia
package developer, the versions or naming can also update unpredictably.
This has frequently caused problems when, for example, Homebrew rearrages
their repository structure or migrates a package from one location to another,
or when OpenSUSE updates the version of their MinGW cross-compiler package
causing all WinRPM packages to rebuild including possible ABI changes.

4. Compiling from source is time consuming and error prone

The build-time dependencies to compile a library are invariably more
complicated than the run-time dependencies to use that library once built.
While most Linux systems will have a C/C++ compiler and build toolchain
available, and it's not too complicated to obtain the Xcode command line tools
on personally-owned Macs, it's not absolutely guaranteed. Users can download
and run Julia without having any build tools installed. And for libraries that
require additional less-ubiquitous build tools like cmake or gfortran, BinDeps
does not help with checking or ensuring they are installed.

5. Declarative, macro-based Julia API is difficult to understand and debug

Because of the way library dependencies are declared in BinDeps, and since the
actual installation code is macro-generated, it can be hard to reconstruct why
BinDeps chooses to execute some actions but not others at install time. If
anything goes wrong, there are a few options like trying `BinDeps.debug` and
seeing if a library can be `dlopen`ed, but these are not always informative
enough to diagnose problems.

## Goals for a BinDeps Replacement

Due to the above problems and the technical debt of the existing BinDeps code
and workflow, I propose a ground-up redesign and replacement, with the
following design goals in mind:

1. Always use precompiled binaries on the 3-4 major platforms

The top platforms by Julia users are `x86_64` (aka amd64) Windows, `x86_64`
GNU/Linux (meaning specifically a Linux distribution that uses glibc), `x86_64`
MacOS, and `i686` Windows (often running a 32 bit Julia executable on 64 bit
Windows). Other platforms like `i686` Linux, 32 and 64 bit ARM, Power, FreeBSD,
and non-glibc Linux variants are far less commonly used. The numbers may have
changed somewhat, but based on downloads of Julia binaries from May to October
2015, the approximate relative ratios are:

    Win64   40
    Linux64 29
    Mac64   26
    Win32   15
    Linux32  1

If package authors can build portable binaries for each of the top 4 platforms,
then we can avoid all the complication of relying on external package managers.

2. Completely decouple user install code from developer build code

To minimize the number of moving parts and possible things that can go wrong
for users when installing a package, the code that runs at install time to
obtain a binary should be as simple and do as little as possible. All the Julia
package would require is a mapping from platform (specifically the GNU triple
used to build Julia, recorded in `Sys.MACHINE` from the output of
`gcc -dumpmachine`, in order to distinguish glibc-using Linux distributions
from those that use a different libc such as musl in Alpine Linux) to url and
checksum. This mapping could be recorded in a json or toml file, or a Julia
dictionary. We can standardize to a single archive format (most likely
`.tar.gz`) and expected internal layout of the packaged binaries. The install-
time code would download an archive, verify its checksum, and extract it.
Everything else should be moved to separate code that is only used at binary
build time by the package developer (or potentially as a fallback on exotic
platforms that the package developer does not provide binaries for).
Users only ever need to execute simple minimal "BinaryDownloader" code,
package developers use the more complicated set of "BinaryBuilder" tooling.
As a longer-term goal if this design works well in practice, we could consider
deprecating and eventually removing the execution of arbitrary code in
`deps/build.jl` at package install time. The BinaryDownloader functionality
could possibly be incorporated as a built-in component of the Julia package
manager.

3. Allow package authors to cross-build binaries from anywhere

It can be very difficult to build and test code for a platform you don't
personally use, especially if your only options for development and testing
are remote CI services. We should provide a sandboxed, controlled,
standardized build environment that anyone can use to build binaries for the
major platforms. Once things are working properly the build code should be
easy to deploy and automate on CI services like Travis, AppVeyor, Circle CI,
etc, but there should be a local development mode that works interactively to
get things working initially. The edit-build-debug cycle is much faster in a
local terminal than iterating through remote CI builds. Setting up a local
build environment for package developers should ideally not require root
privileges either, for the same reasons we want to avoid requiring them to
install a package.

## Platform Specific Notes

1. Linux

The trick to making generic Linux binaries that work across many different
(glibc-using) distributions is to build from the oldest environment any of
your users will be running. If you include all dependencies with your binary
that aren't guaranteed to already be present on users' systems, then
substituting a newer version of a system library at run time usually works
thanks to symbol versioning. This is how Conda and Python
[manylinux](https://www.python.org/dev/peps/pep-0513/) wheels work. Since the
build environment is very old, typically the oldest CentOS version that is
still supported, you often need to obtain a newer compiler version to build
libraries such as LLVM that require recent features like C++11 support.

While we could use Docker to provide this environment, it has multiple
downsides and is not necessary for what we need. Docker is well supported on
CI services, but it's not possible to use locally without root privileges,
and cannot be run by package authors who use Home versions of Windows except
via a VM. Docker encourages creating large monolithic images that can only be
consumed by Docker. Sandboxing a controlled build environment can use simpler
tools such as [PRoot](https://proot-me.github.io/). The following steps should
work on any `x86_64` Linux system without needing root (try it on JuliaBox):

    curl -LO https://github.com/proot-me/proot-static-build/releases/download/v5.1.1/proot_5.1.1_x86_64_rc2
    chmod +x proot_5.1.1_x86_64_rc2
    mkdir centos6
    curl -L https://github.com/CentOS/sig-cloud-instance-images/raw/CentOS-6/docker/centos-6-docker.tar.xz | tar -C centos6 -xJf -
    cd centos6
    ../proot_5.1.1_x86_64_rc2 -R .
    cat /etc/centos-release

For build tools like recent toolchain versions, we should dogfood the system
and have it self host by packaging and consuming its own artifacts rather than
baking them into a Docker image. We can allow using Docker as an execution
backend for build scripts, but it shouldn't be the only option.

2. Windows

The recommended approach for building Windows binaries will be via cross
compilation. The Windows binaries of Julia itself are cross compiled from
Cygwin, but for the sake of automation and familiarity for package authors,
it will be better to provide a more unified build environment and instead
cross compile package binaries from Linux. There are many variations in how
MinGW-w64 gcc cross compilers can be configured in terms of exception handling
and threading models, and C++ ABI settings. We will have to do some testing
to identify and provide a recommended cross compiler configuration and
corresponding binaries. WinRPM packages are built using the OpenSUSE MinGW-w64
cross compiler package, but since that upgraded to GCC 7 while Cygwin's cross
compiler is still on GCC 5, there are currently some incompatibilities that
need to be resolved.

3. MacOS

Mac is the most difficult to build for from other platforms. Apple's license
agreements do not allow virtualizing the OS or using their SDK on non-Apple
hardware. There are working
[Vagrant boxes](https://app.vagrantup.com/jhcook/boxes/osx-yosemite-10.10)
available that we can use for sandboxing and environment isolation. Travis CI
can be used for automated builds. Perhaps [Darling](https://github.com/darlinghq/darling)
can help for package authors who don't have local access to a Mac.

## Proposed Workflow

1. Depending on where the BinaryBuilder tool is executed from, use one of
   - PRoot (executing from Linux)
   - Docker (if installed and user has privileges)
   - Vagrant

   to launch standardized minimal build environment into a terminal.

2. Instruct package author (in shell message-of-the-day and documentation)
   to install the build dependencies for the library they want to build into
   the environment.

3. Package author downloads source code of desired library, runs its build
   system and installs binaries into a designated output prefix.

4. Package author exits build environment. Shell history is saved as a script
   to be used for future execution and CI deployment. Artifacts from output
   prefix are saved into an archive for uploading. Archive should also
   include record of build commands that were used, for reproducibility.

5. Repeat process in modified environment for each new platform. Environment
   presets `CC`, `--host` flags for autotools via `config.site`, etc.

6. Upload all binary artifacts to github releases of the Julia package, or an
   auxiliary repository for saving build scripts and running on CI.

7. Save mapping of platform to binary urls and checksums into the Julia
   package.

## Open Questions

- The above workflow has mostly been considering the use case where a single
Julia package needs a single binary download, and the versioning is coupled.
A given version of the Julia package would depend on a particular version of
the binary download. A question is if multiple packages want to depend on a
common library, do we incorporate a binary level of version resolution into
the package manager?

- A single source package and build script may make sense to distribute as
independent subpackages. For example a gcc compiler build can have separate
C, C++, Fortran, etc compiler components which could be installed
independently. And the runtime library could be packaged separately from the
compiler executable.

- If an existing package manager already builds and provides binaries that
meet all of our requirements, we may want to repackage and mirror their work
instead of duplicating it. We could make a copy of a Homebrew bottle, or a
WinRPM or Conda package, rearrange the structure and repackage if needed to
fit the expectations for our archives, then upload permanently to a location
that we have control over. If their binaries work then we can use them, the
main thing to avoid is having updates that are outside of our control break
something that used to work. It would also simplify matters to avoid the
install-time dependency on an entirely separate package manager implemented
in a different language (ruby for Homebrew, python for conda).
