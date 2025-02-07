Building and Installation
=========================

To build K-Michelson, please follow the instructions below. Note that:

- all commands should be executed in the K-Michelson software archive root
- only Linux is officially supported; other systems may have limited support

K-Michelson has two direct dependencies:

1. [K framework](https://github.com/kframework/k) - for building the K-Michelson
   interpreter and executing Michelson code using the K-Michelson interpreter
2. [Tezos client](http://tezos.gitlab.io/index.html) for cross-validation of
   K-Michelson test execution against test execution on the reference Michelson
   interpreter.

To simplify installation, we package pinned versions of (1-2) under the `/ext`
directory. As part of the installation process, we must first build these
dependencies.

Installing the Prerequisites
----------------------------

The K framework and Tezos client each have a number of their own dependencies
that must be installed before they can be built. On an Ubuntu system, do the
following to install all needed dependencies:

```sh
sudo apt-get install rsync git m4 build-essential patch unzip bubblewrap wget  \
pkg-config libgmp-dev libev-dev libhidapi-dev libmpfr-dev flex bison           \
maven python3 cmake gcc zlib1g-dev libboost-test-dev libyaml-dev               \
libjemalloc-dev openjdk-8-jdk clang-8 lld-8 llvm-8-tools pcregrep cargo
```

Note that the JDK and Clang packages referenced above typically can be
substituted with more recent versions without any issues.

K-Michelson requires Z3 version 4.8.11, which you may need to install from a
source build if your package manager supplies a different version. To do so,
follow the instructions
[here](https://github.com/Z3Prover/z3#building-z3-using-make-and-gccclang) after
checking out the correct tag in the Z3 repository:

```sh
git clone https://github.com/Z3Prover/z3.git
cd z3
git checkout z3-4.8.11
python scripts/mk_make.py
cd build
make
sudo make install
```

You will also need a recent version of
[Haskell stack](https://docs.haskellstack.org/en/stable/install_and_upgrade).
You can either install the Ubuntu package and upgrade it locally:

```
sudo apt-get install haskell-stack
stack upgrade --binary-only
```

or else get the latest version with their installer script by doing:

```
curl -sSL https://get.haskellstack.org/ | sh
```

You will additionally need to install a recent version of the
[OCaml package manager, opam](https://opam.ocaml.org/doc/Install.html). For
Ubuntu, the recommended installation steps are:

```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:avsm/ppa
sudo apt-get install opam
```

For other Linux distributions, you may need to adapt the above instructions,
especially package names and package installation commands.
Consult your distribution documentation for details as well as the links
above for guidance on installing Haskell Stack as well as opam.

Building K-Michelson
--------------------

Build the K Framework (if you don't have a global install) and Tezos
dependencies:

```sh
git submodule update --init --recursive
make deps
```

The following command will build all versions of the semantics including:

- The LLVM backend for running Michelson unit tests (`.tzt` extension);
- The Haskell backend for running symbolic Michelson unit tests (also `.tzt`
  extension);
- The compatibility layers for validating the semantics against the Tezos
  reference client.

```sh
make build -j8
```

Running Test Suites
-------------------

There are three major test-suites you may be interested in running.

The unit tests (running individual `*.tzt` programs and checking their exit
code):

```sh
make test-unit -j8
```

The symbolic unit tests (running individual `*.tzt` programs and checking their
output using `lib/michelson-test-check`). Note that their are three flavours of
symbolic unit tests. Those with the `stuck.tzt` extension, test ill-formed
programs and are expected to get stuck before completing exectution. Those with
the `fail.tzt` extension, are "broken" tests where assertions are expected to
fail.

```sh
make test-symbolic -j8
```

The validation tests against the Tezos reference client:

```sh
make test-cross -j8
```
