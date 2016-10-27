# JULEP 3

- **Title:** Pkg3
- **Authors:**
  - Stefan Karpinski <<stefan@karpinski.org>>
  - Art Diky <<wildart@gmail.com>>
- **Created:** October 21, 2016
- **Status:** work in progress

## Abstract

Pkg3 is the working name of a replacement for Julia's built-in package manager, unofficially known as Pkg2. Pkg2 was introduced in Julia 0.2 as a replacement for Pkg1, the original package mangager included in the 0.1 release of Julia.

## Rationale

There are a number of issues with the design of Pkg2, which necessitate a redesign:

- Pkg2's METADATA repository format uses many small files to represent data, which leads to awful performance on many filesystems, especially on Windows.
- Pkg2 uses a variety of ad hoc configuraration formats which are simple but not particularly consistent.
- Pkg2 identifies versions of packages by git SHA1 commit hashes. This forces the package manager to use git to acquire package versions and makes package installation and verification impossible without including the entire git history of a package impractical.
- Some Julia packages have large objects in their git history, which users are forced to download even when they are installing more recent versions which no longer include those objects.
- Pkg2 makes replacing a package with another package of the same name with disjoint git history a nightmare. This happened when `Stats` was renamed to `StatsBase` and a new `Stats` package was created. The only practical way to resolve this situation was to delete all packages and start over.
- Pkg2 was designed to allow package development in the same location as package installation for usage. This design forces Pkg2 to use complex and subtle heuristics to try to determine when it is safe to update or modify installed packages. A large amount of code complexity stems from this design.
- Pkg2's package version resolution is designed to depend only on requirements and version information in METADATA. This implies that any update tends to update all packages, which is typically undesirable, and effectively assumes that the user has carefully and accurately curated their exact requirement of packages, and that package developers never break things – neither of which is typically true.
- In Pkg2 any operation on packages invokes a full version resolution: adding or removing a new package updates all packages. This is very bad behavior for a package manager. It should be possible to add a new package with zero or minimal changes to pre-installed packages. It should always be possible to remove a package by simply removing all packages that depend on it.
- Pkg2 provides little support for projects tracking the precise versions of libraries and packages that they have used. This makes reproducibility more challenging than it should be.
- The `JULIA_PKGDIR` environent variable allows some amount of simulation of virtualenv-like "environments" – i.e. different sets of packages and language versions. This could be much better supported, however, and environment contents should ideally be easily commitable and sharable between different projects and systems, at various levels of granularity.

## Depots

A **depot** is a file system location where we keep infrastructure related to Julia package management: registries, libraries, packages, and environments. There are typically at least three of these:

- **Standard depot:** default packages and libraries that ship with a specific version of Julia. This depot is strictly read-only. These versions of libraries and packages serve as a fallback when no other depots available. If you delete or disable this depot as well, standard packages will be unavailable. Example: `/usr/local/share/julia/standard`.

- **System depot:** package versions and libraries installed here are available to everyone on the system. They are typically only writable by administrators. If users want to add or upgrade packages, they will do so in their individual user depots. Example: `/usr/local/share/julia/system`.

- **User depot:** package versions and libraries installed by a user. Example: `~/.julia/`.

Note the lack of Julia versions in this scheme: a depot is expected to be shared between different Julia versions. This should work because of the principle of immutability (see below): since we don't update versions of libraries or packages in place, installed copies can be shared between different versions of Julia without issues. Different sets of library and package versions are handled at the environment level.

Each package depot contains the following directories:

- **`registries`:** named registries describe sets of packages, versions and compatibility between them.
- **`libraries`:** installed versions of libraries (e.g. `libcairo`,  `libpango`).
- **`packages`:** installed versions of Julia packages (e.g. `Cairo`,  `DataFrames`, `JuMP`).
- **`environments`:** named sets of versions of libraries and packages and global configuration.

Some environment and/or Julia variable – `DEPOT_PATH` maybe? – will control the set of depots visible to a Julia process. The registries, libraries, packages, and environments visible to Julia are the union across all depots in the depot path.

The set of registered packages visible to a Julia process is the union of all packages specified across all registries, merging specifications of the same package occurring in multiple registries by the following rules:

- The set of known packages is the union across all registries.
- The set of available versions of a package is the union across all registries.
- If the same version of a package appears in multiple registries, all versions must match.
- The registry with the largest registered version of a package determines its metadata;
  - If two different registries "tie" then the package metadata must match.

The set of installed library versions is the union across depots. If the same library version occurs multiple times in the depot path, the first occurance is used – different instances of the same library version may be different depending on how they are configured and installed. The set of installed package versions is the union across depots. If the same package version occurs multiple times in the depot path, the first occurance is used. If installed correctly, different installations of the same package should be identical.

Each named environment specifies a set of specific library and package versions. These libraries and packages do not need be installed in the same depot where the environment appears. They can be provided by another package depot, allowing preinstalled libraries and packages to be "inherited" from a system depot, for example. The default environment name is `v$(VERSION.major).$(VERSION.minor)`. This allows different versions of Julia to have different default environments.

## Immutability

Installed libraries and packages are immutable: instead of updating libraries or packages in-place, once they are successfully installed, Pkg3 leaves them as-is until they are no longer needed. This requires a "cleanup" mechanism that does garbage collection of old, unused versions of libraries and packages. To that end, `Pkg3` will maintain a sorted `~/.julia_env.log` file tracking the paths of environment files they have used. During cleanup, if a path no longer points to a valid environment file, the entry is removed from `~/.julia_env.log`; if a path does point to a valid environment file, it is retained, and library and package versions referred to by it are considered to be in use. Any library or package versions that are not marked as in use are removed. When cleaning up a system depot, all user environment logs are scanned; when cleaning up a user depot, only that user's environment log is considered.

## Environments

An **environment** catures a specific set of package and library versions and their global configuration. Pkg2 has some limited support for changing environments using the `JULIA_PKGDIR` environement variable. Pkg3 makes named environments and project-local environments a primary part of its design, making the invocation of Julia with different sets of libraries and packages for more convenient. It also standardizes how to record the names and versions of libraries and packages that are used, improving reproducibility.

In Pkg2, package operations like `Pkg.add`, `Pkg.rm`, and `Pkg.update` are somewhat inconsistent about whether they operate on the current running Julia process or not. This is because different actions have different feasibility with respect to the current session: it's possible to install or update a package before it is loaded, but it is impossible to remove or update an already-loaded package. Thus, performing operations on the set of available packages *in general* requires a restart of the process before it can take effect, but installing and then loading a new package without restarting the current process is common and useful.

In Pkg3, general operations on environments are not done in the Julia process using an environment. Instead, they are done through a standalone process, which (although it is implemented in Julia) does not operate within the environment that it manipulates. The most common operation, however – installing and loading a new package – will typically be done implicitly and automatically in an interactive Julia session. In other words, when the user does `using XYZ` in the REPL, if  `XYZ` is not installed, the REPL will promt the user if they want to install `XYZ` and its dependencies, and if they agree, it will install and then load it. Since this is the most common operation it can be done without restarting the current Julia process, it makes sense that it be handled specially. When the user wants to remove package or update packages from an environment, they will instead invoke an external package management mode (`julia --pkg`?), which makes it clear that changes will not affect any currently running Julia sessions. The impact on usability is a strict improvement:

- Adding packages and loading them is easier since one simply does `using XYZ` and answers interactive prompts.
- Removing and upgrading packages is no less difficult since it previously required restarting the current Julia process anyway, and is less confusing since the requirement to restart is explict since running a separate process clearly doesn't affect the current one.

### Using Environments

When starting Julia, it is given an environment by default, by name or by path:

- `julia`: use the default named environment – `v$(VERSION.major).$(VERSION.minor)`.
- `julia --env=abc`: use the environment named "abc", searched for in the depot path.
- `julia --env=.`: use the local project environment (see below).
- `julia --env=./proj`: use the project environment of the directory `./proj`.
- `julia --env=./env.toml`: use environment described by the file `./env.toml`.

An environment spec with no slash is taken to be a named environment – except for the special name `.` which indicates using the current project environment. An environment spec with a slash is taken to be a path (relative or absolute): if the path is a directory, it is interpreted as a project and the project environment is used; if the path is a file, it is loaded as an environment specification (in TOML format, see "Configuration" below).

An environment spells out exactly what version of each of a set of packages and libraries to use (version, hash, path, etc.). A Julia process can be "open" or "closed" with respect to its environment:

- **Open:** packages that are not in the environment can be loaded. They will be resolved greedily in the order they are loaded, choosing the highest installed version that satisfies the requirements of the environment and all loaded packages. If no statisfactory version is installed, but some registered version exists that would satisfy all requirements, the user is prompted to install and use it.
- **Closed:** packages that are not in the environment cannot be loaded.

By default, Julia runs in open mode. When testing or deploying, however, Julia should default to closed mode to help ensure that a project hasn't inadvertently used packages that aren't recorded as dependencies. Since the project configuration also records which packages are direct dependencies, closed mode could enforce that project code only uses direct dependencies and indirect dependencies are only loaded indirectly. Note that this also helps address the problem that different packages may refer to different packages by the same top-level name.

### Project Environments

The environment specification of a project is split into three files: `Config.toml`, `Manifest.toml`, and `Local.toml`. (Each file name may also be prefixed with `Julia`, in which case the non-prefixed file, if it exists, is ignored.) The purpose of these files is to separate the environment into three parts:

- `Config.toml`: manual configuration, checked into version control (intput)
- `Manifest.toml`: generated information, checked into version control (output)
- `Local.toml`: generated information, not checked into version control (by product)

Accordingly, `.gitignore` for Julia projects should include entries for `/Local.toml` and `/JuliaLocal.toml` so that those files are ignored by version control. The `Config.toml` file controls what subset of environment information goes into `Manifest.toml` versus what goes into `Local.toml` – everything ends up in one or the other. Examples of different scenarios with various choices of manifest subsets:

- A project meant to run on a single system (or homogenous systems) may choose to save everything in the manifest, including exact versions of packages and libraries, paths to them, even hashes of them, so that a complete record is checked into the project repository.
- A project meant to run on different systems, on the other hand, may choose to check specific project versions and hashes into version control, but not library information, using libraries available on each system.
- Published packages will generally not check specific dependency versions into version control since these will differ among developers and users. They will, however, check in general dependency version requirements (e.g. `XYZ = "1.2-1.9"`). During early development, however, it may be desirable to check in more detail so that different developers can stay in sync more easily.

When using the current project environment, specified by starting Julia with the ` --env=.` flag, the project directory is searched for by looking in the current directory and each parent directory for `JuliaConfig.toml` or `Config.toml`. If a directory is found containing a file by this name, it is considered to be the project root and the config, manifest and local files are loaded from there.

## Packages

Packages continue to work much as they have previously with a few exceptions:

1. Each package has `Config.toml` and `Manifest.toml` files.
2. `Config.toml` contains an entry giving the package a [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier).
3. Package versions are identified by a hash of a source tree instead of a git commit.
4. Eventually packages will not need to be git repositories.

UUIDs for registered packages will be assigned and when new packages are generated, a UUID will be created (this should happen even for private, unregistered packages). UUIDs will generally not be user-facing, but they are used internally to identify packages in registries and environment files. The purpose of UUIDs is to allow renaming of packages and moving of packages between different registries. A couple of scenarios to consider before arguing against using UUIDs:

- The `Stats` / `StatsBase` situation: `Stats` was renamed to `StatsBase` and a new package also called `Stats` was created. This broke many people's package installations and cause a great deal of grief. With packages identified by UUID, this kind of rename is completely unproblematic.
- Two different packages may be created in different private registries with the same name. If these are both later made public, they may need to be renamed, but some way of knowing which one an old environment using one of them was referring to. Version hashes should be unique, but environments can record unregistered states of packages: unless every tree hash that could ever have been recorded in an environement using a package is known, it's impossible to figure out which package was used. If packages have UUIDs and these are recorded in environments, then it will always be possible to know which package was meant.

Identifying package version by hashes of *source trees* rather than git commit hashes allows us to acquire and verify package versions without necessarily using git, and even with git it makes it easier to support shallow cloning and history rewriting, as long as the source trees of a published version doesn't change. The git style SHA1 tree hash is one means of identifying a source tree, but we may want to support other hashes since SHA1 is no longer considered secure. We could, for example, also publish SHA2-512 hashes for the source trees of package versions, along side SHA1 hashes, allowing smooth transitioning to a more secure hash. With multiple coexisting ways of acquiring package versions, we can also smoothly transition away from using git alone for delivery of package code.

## Registries

A **registry** is a Pkg3 replacement for the METADATA repository. Crucially, Pkg3 supports using multiple registries, and there will be "cathedral" and "bazaar" style public registries, and private registries will be supported. Private registries allow organizations to internally register private packages and versions which can refer to and depend on public packages. Registries provide four kinds of information:

1. Bidirectional many-to-many mapping between package names and UUIDs.
2. A list of versions for each package, identified by their source tree hash.
3. Version dependency and compatibility information.
4. Where to get each package version.

The latest UUID associated with a name is the current one; other UUIDs were previous packages associated with that name. A UUID may have multiple names associated with it over time, but the latest one is current. If the same name occurs in different registries, referring to different UUIDs, then there is a name conflict which must be resolved interactively as needed. For example, if a user asks to add `XYZ` but the name refers to different packages in different registries, then the user should be prompted for which one they want.

Each version is associated with a specific source tree, unlike Pkg2 where each version is associated with a git commit. This allows us to acquire and verify package versions without necessarily using git, and even with git it makes it easier to support shallow cloning or history rewriting, as long as the source trees of published versions don't change. The git style SHA1 tree hash is one means of identifying a source tree, but we may want to support other means since SHA1 is no longer considered secure. We could, for example, also publish SHA2-512 hashes for the source trees of package versions, thereby allowing them to be securely verified even though SHA1 is no longer secure.

## Versions & Compatibility

Expressing compatibility between various versions of packages is complicated by the fact that compatibility claims for a particular version can either be:

- mistakenly incorrect when published, or
- correct when published but so broad that they later become incorrect.

Pkg2 allows and even encourages very loose dependency declarations and deals with both of the above situations by allowing compatibility claims to be adjusted after the fact. Dependencies can and are expected to be changed in METADATA to adjust for mistakes and invalidation. This causes significant complexity and confusion, however: the dependencies of a package version according to its own immutable source may not match the current dependencies registered for it in METADATA – which are still potentially evolving. Because of this, Pkg2 contains tricky logic about which compatibility claims take precedence – those in the source tree or those in METADATA. These rules are especially complicated since Pkg2 supports development of packages where they are installed, further muddying what the definitive record of compatibility is.

In Pkg3, a package version's compatibility claims are immutable. While compatibility claims may still be incorrect, they cannot be changed, only superseded by a newer version. Overly broad compatibility claims cannot, by design, be expressed in the first place. In this design, any invalidation of claimed compatibility can only stem from another package's failure to follow [semantic versioning](http://semver.org/) correctly. However, since this will certainly occur in practice, when it does, it can easily be remedied by a new release of either package. Instead of modifying compatibility claims after the fact, if a version's claims are found to be mistaken, one publishes a patch, with the same major and minor version numbers, with corrected dependencies. Pkg3's version resolution will favor the most recent patch very strongly: unless you explicitly ask for an earlier patch specifically, a freshly installed or updated package will always be the latest patch in its major-minor series. Package developers should follow semantic versioning strictly and *only* include bug fixes in patch releases: patches should neither break existing features nor introduce new features.

Compatibility claims in Pkg3 are expressed at *exactly* minor version granularity. This may be easiest to explain starting with the textual form. In configuration files, sets of compatible versions are expressed using arrays of string literals (in TOML format), each string being of one of the following forms:

- **minor version:** `"a.b"` includes versions with `major == a && minor == b`;
- **version range:** `"a.b-a.c"` includes versions with `major == a && b ≤ minor ≤ c`;
- **negated patch:** `"!a.b.c"` excludes versions with `major == a && minor == b && patch == c`.

A list of terms expresses a set of package versions: the union of versions included in minor version strings and version range strigs, minus the specific versions excluded by negated patch strings. In other words, the version list `["1.2-1.4", "!1.2.5", "2.0"]`  includes any version such that

```julia
major == 1 && (2 ≤ minor ≤ 4) && !(major == 2 && patch == 5) || major == 2 && minor == 0)
```

Compatibility lists should be normalized according to the following rules:

- versions and ranges should be mutually disjoint;
- versions and ranges should appear in sorted order by major and minor version;
- versions and ranges which can be coalesced should be combined into a single range;
- negated patches should follow the version or range in which they are contained, separated from it only by smaller negated patches (i.e. negated patches are sorted by major, minor and patch numbers).

Following these rules, each possible set of compatible versions can be expressed in exactly one way. Here are some examples of normalized version sets:

```toml
["1.2"]
["1.2", "!1.2.5"]
["1.2-1.3", "!1.2.5"]
["1.2-1.4", "!1.2.5", "2.0"]
["1.2-1.4", "!1.2.5", "!1.4.0", "2.0"]
["1.2-1.4", "!1.2.5", "!1.4.0", "2.0-2.1"]
["1.2-1.4", "!1.2.5", "!1.4.0", "2.0-2.5", "3.0"]
```

Compatibility sets include an unbounded number of potential future patches, but include a finite number of minor versions. A package should not declare compatibility with a minor version series unless some version in that series has actually been published – this guarantees that compatibility can (and should) be tested. If a new compatible major or minor version of a package is released, this should be reflected by publishing a new patch that expands the compatibility claims. If a new patch of an otherwise compatible major/minor version series contains a bug that breaks compatibility, a new patch of each package should be released: a patch of the buggy package, fixing the bug, and a patch of the other package, excluding the buggy version from its compatibility claims.

## Configuration

Pkg3 uses [TOML](https://github.com/toml-lang/toml) for configuration files. Several other projects have adopted this format: see [Cargo](http://doc.crates.io/manifest.html) and [PEP 518](https://www.python.org/dev/peps/pep-0518/) for example. This [format comparison](https://github.com/toml-lang/toml#comparison-with-other-formats) has some thoughts and justifications for using this format over other common configuration formats. The basic justification is:

- compared to **JSON** it is more human readable and writeable
- compared to **YAML** it is far simpler to parse and understand
- compared to **INI** it is very similar but standardized
- compared to **XML** it is… hah, no.

All said, TOML seems to be the most reasonable format for simple, human-readable configuration files. An implementation of TOML parsing and printing in Julia can be found [here](https://github.com/wildart/TOML.jl). There are a few other implementations floating around, and this version need not be the one we adopt, but it has been used for experimentation during the design process so it should handle formats discussed in what follows.

### Configuration Fragments

We'll begin by describing certain types of configuration fragments. Environments and registries use these fragments in similar ways. TOML headers are absolute, not relative, which makes describing fragments a bit awkward. To address this, consider sections to be implicitly relative: if a fragmen has a header `[header]` consider it relative to wherever it occurs, so if that fragment were used in a section called `[section]` then the header would actually be `[section.header]`.

#### Package metadata

High-level description of a package: its UUID, name, license, authorship, where to get it, etc. This will appear in a package's configuration file and copied into any registries that the package appears in.

```toml
name = "Example"
uuid = "86d33384-d511-4271-be88-8c3e434c707e"
license = "MIT"
authors = [
    "Jane Q. Programmer <jane@example.com>",
    "Jack X. Developer <jack@example.com>",
]
description = "Example package."
keywords = ["example", "fake", "unreal"]
documentation = "https://docs.github.io/Example.jl"
homepage = "https://example.com/Example.jl"
repository = "https://github.com/ExampleOrg/Example.jl"
```

#### Version metadata

This descripes a particular version of a package.

```toml
version = "1.2.3"
SHA1 = "739ea886f7ae45ef27f7c0a2ea2bc25d59d40fd2"
SHA2-512 = """
45d8153f80a301a890d5da67592ddf42fb96c4cd3945998386d0293dcf80b44d
c9c8499c6e1ba4068381ac5bb243561de3e9c25e8989e949d56e8438085a9a22
"""
```

Note that the string for a SHA2-512 hash value is allowed to contain extra whitespace including a newline. This improve readability of files including long hash values by avoiding overly long lines. The hash value is a hash of the source tree, computed as trees are hashed in git, but using different hashing functions. Thus, the SHA1 tree hash is the same as the tree name in git, allowing us to retrieve the source version.

#### Compatibility

The compatibility section expresses which libraries and packages a project directly interacts with, either as requirements or "optional dependencies" – i.e. packages that this package has some special code for, only to be loaded if that other package is also loaded. Only direct dependencies and optional packages are specified in the compatibility section. Any indirect dependencies are strictly the concern of the packages that depend on them. Thus, if `Required` depends on `Indirect`, we cannot constrain the version of `Indirect` here, although `Required` can. Thus, if a new version of `Required` comes out that don't use `Indirect` anymore, and we upgrade to that, the package manager is free to get rid of `Indirect`.

```toml
[library.libXYZ]
uuid = "994d35e9-862f-42c9-aa51-d40fef54ab41"
versions = "2.3-2.5"

[package.Required]
uuid = "85241492-0f92-400a-8719-bdc0424991f7"
versions = ["1.2-1.3", "!1.2.5"]

[package.Optional]
uuid = "f7faa14e-633f-4b87-8f63-428f7e99170d"
versions = "3.7"
optional = true
```

The last component of the header is the library or package name, while the `uuid` field gives its UUID – this unambiguously identifies the package. The name is what the local project will refer to and load the package or library as – this should probably match what its published as, although we may want to allow publishing under multiple names simultaneously. The `versions` field is either a string or an array of strings which specifies a set of compatible versions, as described in "Versions & Compatibility" above.

#### Runtime Configuration

Runtime configuration sections allow projects to set global configuration flags to be passed to libraries. This section only makes sense at a project level since there can only be one source of configuaration for a given library or package – i.e. libraries and packages cannot configure other libraries or packages.

```toml
[library.libXYZ]
backend = "abc"
knob = 1.5

[package.Required]
numbers = [4, 8, 15, 16, 23, 42]
```

#### Manifest

The manifest fragment records all the details of which libraries and packages are included in a Julia environement. The information should be kept by running Julia process so that we can save it to a manifest file. Not all of the data will be appropriate to be committed for all kinds projects, so these data may be split between different files – some to be checked into version control and some strictly local.

```toml
[library.libXYZ]
version = "2.3.4"
path = "/home/user/.julia/libraries/libXYZ/2.3.4"
mtime = 2016-10-20T18:28:56.299
CRC32C = "3ba18fe1"
SHA1 = "d2672146a1aca6023073074d765a32d7eb298baf"
SHA2-512 = """
981702a057faa649b7fa24337a67e0d6e8af258f81d0ed8ce90775cdfe0942c6
d18ce0b5747e5fb1123cceb65b1074a9ba20f788e7cbacc7e824bac043f80208
"""

[package.Required]
version = "1.2.8"
path = "/usr/julia/system/packages/Required/1.2.8"
mtime = 2016-10-20T18:29:55.605
CRC32C = "d1a6296e"
SHA1 = "982d4e4e0f728e7e0416472ffb394250c7afd1aa"
SHA2-512 = """
f991d247834effca8ce7114b7100d191d259abf36bbe6a1cf03382a8e1a51171
0c107a0a7b5a5dd21cfd304e7e5525fc2287cc255de15f1c7d4f33ac86990e85
"""

[package.Optional]
version = "3.7.2"
path = "/home/user/.julia/packages/Optional/3.7.2"
mtime = 2016-10-20T18:35:29.124
CRC32C = "595b180b"
SHA1 = "ff1ca382d0f905ce9e75fc829cfa4419123c0491"
SHA2-512 = """
904b16f8cea76f8feb04526983a42a4b11194a840223976497f85e59c0948c3c
3a4ad1c0c5f1b7f61734f4f8cfee74869693fe6be56e56ca9e54398e3ea06765
"""

[package.Indirect]
version = "1.5.3"
path = "/usr/julia/system/packages/Indirect/1.5.3"
mtime = 2016-10-21T10:42:25.366
CRC32C = "2ffefb96"
SHA1 = "8182d2ea3d4427eccc7e968923cb1bf6affb74c8"
SHA2-512 = """
7cc5a55bf2f55f4ce95d4d63594bb5d2c468a41c552eb6c5d29a9ffcb8a8b40f
665b09748acc0cf3af9eeef81f55805269b86e9f26e32ede03c11d2043bf3f2d
"""
```

### Source Package File

Package configuration includes package metadata and compatibility sections for libraries and packages:

```toml
name = "Example"
uuid = "86d33384-d511-4271-be88-8c3e434c707e"
license = "MIT"
authors = [
    "Jane Q. Programmer <jane@example.com>",
    "Jack X. Developer <jack@example.com>",
]
description = "Example package."
keywords = ["example", "fake", "unreal"]
documentation = "https://docs.github.io/Example.jl"
homepage = "https://example.com/Example.jl"
repository = "https://github.com/ExampleOrg/Example.jl.git"

[library.libXYZ]
uuid = "994d35e9-862f-42c9-aa51-d40fef54ab41"
versions = "2.3-2.5"

[package.Required]
uuid = "85241492-0f92-400a-8719-bdc0424991f7"
versions = ["1.2-1.3", "!1.2.5"]

[package.Optional]
uuid = "f7faa14e-633f-4b87-8f63-428f7e99170d"
versions = "3.7"
optional = true
```

### Registry Package File

Each registered package has its own file (name TBD, but probably `Example.toml`), describing the package, all its registered versions, and their compatibility and requirements on other libraries and packages.

```toml
name = "Example"
uuid = "86d33384-d511-4271-be88-8c3e434c707e"
license = "MIT"
authors = [
    "Jane Q. Programmer <jane@example.com>",
    "Jack X. Developer <jack@example.com>",
]
description = "Example package."
keywords = ["example", "fake", "unreal"]
documentation = "https://docs.github.io/Example.jl"
homepage = "https://example.com/Example.jl"
repository = "https://github.com/ExampleOrg/Example.jl.git"

  [[version]]
  version = "1.2.3"
  SHA1 = "739ea886f7ae45ef27f7c0a2ea2bc25d59d40fd2"
  SHA2-512 = """
  45d8153f80a301a890d5da67592ddf42fb96c4cd3945998386d0293dcf80b44d
  c9c8499c6e1ba4068381ac5bb243561de3e9c25e8989e949d56e8438085a9a22
  """

    [version.library.libXYZ]
    uuid = "994d35e9-862f-42c9-aa51-d40fef54ab41"
    versions = "2.3-2.5"

    [version.package.Required]
    uuid = "85241492-0f92-400a-8719-bdc0424991f7"
    versions = ["1.2-1.3", "!1.2.5"]

    [Optional]
    uuid = "f7faa14e-633f-4b87-8f63-428f7e99170d"
    versions = "3.7"
    optional = true

  [[version]]
  version = "1.2.4"
  SHA1 = "e92729c0e7c23d9f83fadba3e197ab9b5ddd9791"
  SHA2-512 = """
  fd22289bb2440e9d6c112ff4b33e36183a792edafb2cd96eb688ef931faddf9c
  81d4a7a544921bc3c5d79aa74db0a163fa8f75f57c6fb603810dd3d51e17ba2e
  """

    [version.library.libXYZ]
    uuid = "994d35e9-862f-42c9-aa51-d40fef54ab41"
    versions = "2.3-2.6"

    [version.package.Required]
    uuid = "85241492-0f92-400a-8719-bdc0424991f7"
    versions = ["1.2-1.4", "!1.2.5", "2.0"]

    [Optional]
    uuid = "f7faa14e-633f-4b87-8f63-428f7e99170d"
    versions = ["3.7", "!3.7.3"]
    optional = true
```

This format is pretty verbose. We could design a custom compression scheme for this format, aggregating information across multiple versions of the same package, or simply use general purpose compression. General purpose compression would be easier, certainly, but would still require parsing of a potentially very large number of version sections once they're uncompressed. A custom compression scheme could support faster parsing of logically compressed data, allowing the package manager to query the compressed data as-is.
