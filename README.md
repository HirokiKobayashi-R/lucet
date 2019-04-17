# Lucet &emsp; [![Build Status]][travis]

[Build Status]: https://travis-ci.org/fastly/lucet.svg?branch=master
[travis]: https://travis-ci.org/fastly/lucet

**Lucet is a native WebAssembly compiler and runtime. It is designed to safely
execute untrusted WebAssembly programs inside your application.**

Check out our [announcement post on the Fastly
blog](https://www.fastly.com/blog/announcing-lucet-fastly-native-webassembly-compiler-runtime).

Lucet uses, and is developed in collaboration with, Mozilla's
[Cranelift](http://github.com/cranestation/cranelift) code generator.

Lucet powers Fastly's [Terrarium](https://wasm.fastlylabs.com) platform.

---

## Status

Lucet supports running WebAssembly programs written in C (via `clang`), Rust,
and AssemblyScript. It does not yet support the entire WebAssembly spec, but
full support is [coming in the near future](#lucet-spectest).

Lucet's runtime currently only supports x86-64 based Linux systems.

## Contents

### `lucetc`

`lucetc` is the Lucet Compiler.

The Rust crate `lucetc` provides an executable `lucetc`. It compiles
WebAssembly modules (`.wasm` or `.wat` files) into native code (`.o` or `.so`
files).

### `lucet-runtime`

`lucet-runtime` is the runtime for WebAssembly modules compiled through
`lucetc`. It is a Rust crate that provides the functionality to load modules
from shared object files, instantiate them, and call exported WebAssembly
functions. `lucet-runtime` manages the resources used by each WebAssembly
instance (linear memory & globals), and the exception mechanisms that detect
and recover from illegal operations.

The bulk of the library is defined in the child crate
`lucet-runtime-internals`.  The public API is exposed in `lucet-runtime`.  Test
suites are defined in the child crate `lucet-runtime-tests`. Many of these
tests invoke `lucetc` and the `wasi-sdk` tools.

`lucet-runtime` is usable as a Rust crate or as a C library. The C language
interface is found at `lucet-runtime/include/lucet.h`.

### `lucet-wasi`

`lucet-wasi` is a crate providing runtime support for the [WebAssembly System
Interface (WASI)](https://wasi.dev).  It can be used as a library to support
WASI in another application, or as an executable, `lucet-wasi`, to execute WASI
programs compiled through `lucetc`.

See ["Your first Lucet application"](#your-first-lucet-application) for an
example that builds a C program and executes it with `lucet-wasi`.

For details on WASI's implementation, see
[`lucet-wasi/README.md`](lucet-wasi/README.md).

### `lucet-wasi-sdk`

[`wasi-sdk`](https://github.com/cranestation/wasi-sdk) is a Cranelift project
that packages a build of the Clang toolchain, the WASI reference sysroot, and a
libc based on WASI syscalls. `lucet-wasi-sdk` is a Rust crate that provides
wrappers around these tools for building C programs into Lucet modules. We use
this crate to build test cases in `lucet-runtime-tests` and `lucet-wasi`.

### `lucet-module-data`

`lucet-module-data` is a crate with data structure definitions and serialization
functions that we emit into shared objects with `lucetc`, and read with
`lucet-runtime`.

### `lucet-analyze`

`lucet-analyze` is a Rust executable for inspecting the contents of a shared
object generated by `lucetc`.

### `lucet-idl`

`lucet-idl` is a Rust executable that implements code generation via an
Interface Description Language (IDL).  The generated code provides zero-copy
accessor and constructor functions for datatypes that have the same
representation in both the WebAssembly guest program and the host program.

Functionality is incomplete at the time of writing, and not yet integrated with
other parts of the project.  Rust code generator, definition of import and
export function interfaces, and opaque type definitions are planned for the
near future.

### `lucet-spectest`

`lucet-spectest` is a Rust crate that uses `lucetc` and `lucet-runtime`, as well
as the (external) `wabt` crate, to run the official WebAssembly spec test suite,
which is provided as a submodule in this directory. Lucet is not yet fully spec
compliant, and the implementation of `lucet-spectest` has not been maintained
very well during recent codebase evolutions. We expect to fix this up and reach
spec compliance in the near future.

### `lucet-builtins`

`lucet-builtins` is a C library that provides optimized native versions of libc
primitives. `lucetc` can substitute the implementations defined in this library
for the WebAssembly implementations.

`lucet-builtins/wasmonkey` is the Rust crate that `lucetc` uses to transform
function definitions in a WebAssembly module into uses of an import function.

### Vendor libraries

Lucet is tightly coupled to several upstream dependencies, and Lucet
development often requires making changes to these dependencies which are
submitted upstream once fully baked. To reduce friction in this development
cycle, we use git submodules to vendor these modules into the Lucet source
tree.

#### Cranelift

We keep the primary Cranelift project repository as a submodule at
`/cranelift`.

Cranelift provides the native code generator used by `lucetc`, and a ton of
supporting infrastructure.

Cranelift was previously known as Cretonne.  Project developers hang out in the
`#cranelift` channel on [`irc.mozilla.org:6697`](https://wiki.mozilla.org/IRC).

#### Faerie

`faerie` is a Rust crate for producing ELF files.  Faerie is used by Cranelift
(through the module system's `cranelift-faerie` backend) and also directly by
`lucetc`, for places where the `cranelift-module` API can't do everything we
need.

### Tests

Most of the crates in this repository have some form of unit tests. In addition,
`lucet-runtime/lucet-runtime-tests` defines a number of integration tests for
the runtime, and `lucet-wasi` has a number of integration tests using WASI C
programs.

### Benchmarks

We created the `sightglass` benchmarking tool to measure the runtime of C code
compiled through a standard native toolchain against the Lucet toolchain. It
is provided as a submodule at `/sightglass`.

Sightglass ships with a set of microbenchmarks called `shootout`. The scripts
to build the shootout tests with native and various versions of the Lucet
toolchain are in `/benchmarks/shootout`.

Furthermore, there is a suite of benchmarks of various Lucet runtime functions,
such as instance creation and teardown, in `/benchmarks/lucet-benchmarks`.

## Development Environment

### Operating System

Lucet is developed and tested on Linux. We expect it to work on any POSIX
system which supports ELF.

Experimentally, we have shown that supporting Mac OS (which uses the Mach-O
executable format instead of ELF) is possible, but it is not supported at this
time.

### Dependencies

Lucet requires:

* Stable Rust, and `rustfmt`. We typically track the latest stable release.
* [`wasi-sdk`](https://github.com/CraneStation/wasi-sdk), providing a Clang
  toolchain with wasm-ld, the WASI reference sysroot, and a libc based on WASI
  syscalls.
* GNU Make, CMake, & various standard Unix utilities for the build system.

### Getting started

The easiest way to get started with the Lucet toolchain is by using the provided
Docker-based development environment.

This repository includes a `Dockerfile` to build a complete environment for
compiling and running WebAssembly code with Lucet, but you shouldn't have to use
Docker commands directly. A set of shell scripts with the `devenv_` prefix are
used to manage the container.

#### Setting up the environment

0) The Lucet repository uses git submodules. Make sure they are checked out
   by running `git submodule init && git submodule update`.

1) Install and run the `docker` service. We do not support `podman` at this
   time. On MacOS, [Docker for
   Mac](https://docs.docker.com/docker-for-mac/install/) is an option.

2) Once Docker is running, in a terminal, and at the root of the cloned
   repository, run: `source devenv_setenv.sh`. (This command requires the
   current shell to be `zsh`, `ksh` or `bash`). After a couple minutes, the
   Docker image is built and a new container is run.

3) Check that new commands are now available:

```sh
lucetc --help
```

You're now all set!

#### Your first Lucet application

The `devenv_setenv.sh` shell script ensures the Lucet executables are available
in your shell. Under the hood, these commands are executed in the Docker
container.  The container has limited visibility into the host's filesystem - it
can only see files under the `lucet` repository.

Create a new work directory in the `lucet` directory:

```sh
mkdir -p src/hello

cd src/hello
```

Save the following C source code as `hello.c`:

```c
#include <stdio.h>

int main(void)
{
    puts("Hello world");
    return 0;
}
```

Time to compile to WebAssembly! The development environment includes a version
of the Clang toolchain that is built to generate WebAssembly by default. The
related commands are accessible from your current shell, and are prefixed by
`wasm32-unknown-wasi-`.

For example, to create a WebAssembly module `hello.wasm` from `hello.c`:

```sh
wasm32-unknown-wasi-clang -Ofast -o hello.wasm hello.c
```

The next step is to use Lucet to build native `x86_64` code from that
WebAssembly file:

```sh
lucetc-wasi -o hello.so hello.wasm
```

`lucetc` is the WebAssembly to native code compiler. The `lucetc-wasi` command
runs the same compiler, but automatically configures it to target WASI.

`hello.so` is created and ready to be run:

```sh
lucet-wasi hello.so
```

#### Additional shell commands

* `./devenv_build_container.sh` rebuilds the container image. This is never
  required unless you edit the `Dockerfile`.
* `./devenv_run.sh [<command>] [<arg>...]` runs a command in the container. If
  a command is not provided, an interactive shell is spawned. In this
  container, Lucet tools are installed in `/opt/lucet` by default. The command
  `source /opt/lucet/bin/devenv_setenv.sh` can be used to initialize the
  environment.
* `./devenv_start.sh` and `./devenv_stop.sh` start and stop the container.

## Security

The lucet project aims to provide support for secure execution of untrusted code. Security is achieved through a combination of lucet-supplied security controls and user-supplied security controls. See [SECURITY.md](SECURITY.md) for more information on the lucet security model.

###  Reporting Security Issues

The Lucet project team welcomes security reports and is committed to providing
prompt attention to security issues. Security issues should be reported
privately via [Fastly’s security issue reporting
process](https://www.fastly.com/security/report-security-issue). Remediation of
security vulnerabilities is prioritized. The project teams endeavors to
coordinate remediation with third-party stakeholders, and is committed to
transparency in the disclosure process.
