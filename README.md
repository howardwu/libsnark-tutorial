# libsnark-tutorial

In this library, we will create a simple zkSNARK application using [libsnark](https://www.github.com/SCIPR-Lab/libsnark), a C++ library for zkSNARK proofs. zkSNARKs enable a prover to succinctly convince any verifier of a given statement's validity without revealing any information aside from the statement's validity. This technology has formed the basis for protocols such as [Zcash](https://z.cash), a cryptocurrency that provides anonymity for users and their transactions.

This tutorial will guide you through installing `libsnark`, setting up a development environment, and building a simple zkSNARK application. This library can be extended to support a testing framework, profiling infrastructure, and more.

## Table of Contents

- [Introduction](#introduction)
- [Build Guide](#build-guide)
  - [Installation](#installation)
- [Development Environment](#development-environment)
  - [Directory Structure](#directory-structure)
  - [Compilation Framework](#compilation-framework)
  - [Development Framework](#development-framework)
- [zkSNARK Application](#zksnark-application)
- [Compilation](#compilation)
- [Further Resources](#further-resources)
- [License](#license)

## Introduction

Zero-knowledge proofs were first introduced by Shafi Goldwasser, Silvio Micali and Charles Rackoff. A zero-knowledge proof allows one party, the prover, to convince another party, the verifier, that a given statement is true, without revealing any information beyond the validity of the statement itself. A zkSNARK is a variant of a zero-knowledge proof that enables a prover to succinctly convince any verifier of the validity of a given statement and achieves computational zero-knowledge without requiring interaction between the prover and any verifier. 

zkSNARKs can be used to prove and verify, in zero-knowledge, the integrity of computations, expressed as NP statements, in forms such as the following:

- "The C program _foo_, when executed, returns exit code 0 if given the input _bar_ and some additional input _qux_."
- "The Boolean circuit _foo_ is satisfiable by some input _qux_."
- "The arithmetic circuit _foo_ accepts the partial assignment _bar_, when extended into some full assignment _qux_."
- "The set of constraints _foo_ is satisfiable by the partial assignment _bar_, when extended into some full assignment _qux_."

A prover with knowledge of the witness for the NP statement can produce a succinct proof that attests to the truth of the NP statement. Anyone can then verify this short proof, which offers the following properties:

-   __Zero-knowledge__
    \- the verifier learns nothing from the proof beside the truth of the statement (i.e., the value _qux_, in the examples above, remains secret).
-   __Succinctness__
    \- the proof is short and easy to verify.
-   __Non-interactivity:__
    \- the proof is a string (i.e. it does not require back-and-forth interaction between the prover and the verifier).
-   __Soundness__
    \- the proof is computationally sound (i.e., it is infeasible to fake a proof of a false NP statement). Such a proof system is also called an _argument_.
-   __Proof of knowledge__
    \- the proof attests not just that the NP statement is true, but also that the
    prover knows why (e.g., knows a valid _qux_).

Together, these properties comprise a _zkSNARK_, which stands for a _Zero-Knowledge Succinct Non-interactive ARgument of Knowledge_.

## Build Guide

This repository has the following dependencies, which come from `libsnark`:

- C++ build environment
- CMake build infrastructure
- GMP for certain bit-integer arithmetic
- libprocps for reporting memory usage
- Fetched and compiled via Git submodules:
    - [libff](https://github.com/scipr-lab/libff) for finite fields and elliptic curves
    - [libfqfft](https://github.com/scipr-lab/libfqfft) for fast polynomial evaluation and interpolation in various finite domains
    - [Google Test](https://github.com/google/googletest) (GTest) for unit tests
    - [ate-pairing](https://github.com/herumi/ate-pairing) for the BN128 elliptic curve
    - [xbyak](https://github.com/herumi/xbyak) just-in-time assembler, for the BN128 elliptic curve
    - [Subset of SUPERCOP](https://github.com/mbbarbosa/libsnark-supercop) for crypto primitives needed by ADSNARK

### Installation

* On Ubuntu 16.04 LTS:

        $ sudo apt-get install build-essential cmake git libgmp3-dev libprocps4-dev python-markdown libboost-all-dev libssl-dev

* On Ubuntu 14.04 LTS:

        $ sudo apt-get install build-essential cmake git libgmp3-dev libprocps3-dev python-markdown libboost-all-dev libssl-dev

* On Fedora 21 through 23:

        $ sudo yum install gcc-c++ cmake make git gmp-devel procps-ng-devel python2-markdown

* On Fedora 20:

        $ sudo yum install gcc-c++ cmake make git gmp-devel procps-ng-devel python-markdown

## Development Environment

__This library includes the completed development environment. If you wish to use the provided environment, you may proceed to the [zkSNARK Application](#zksnark-application).__

### Directory Structure

We will create a library with the following directory structure:

* [__src__](src): C++ source code
  <!-- * [__tests__](src/tests): collection of GTests -->
* [__depends__](depends): dependency libraries

Start by creating a `src` directory and nested `test` directory.
```
mkdir src && mkdir src/test
```

Next, create a dependency directory, called `depends`, and add `libsnark` as a submodule.
```
mkdir depends && cd depends
git submodule add https://github.com/scipr-lab/libsnark.git libsnark
```

### Compilation Framework

We will use `CMake` as our compilation framework. Start by creating a `CMakeLists.txt` file in the root directory and initialize it with the following.
```
cmake_minimum_required(VERSION 2.8)

project(libsnark-tutorial)

set(
  CURVE
  "BN128"
  CACHE
  STRING
  "Default curve: one of ALT_BN128, BN128, EDWARDS, MNT4, MNT6"
)

set(
  OPT_FLAGS
  ""
  CACHE
  STRING
  "Override C++ compiler optimization flags"
)

option(
  MULTICORE
  "Enable parallelized execution, using OpenMP"
  OFF
)

option(
  WITH_PROCPS
  "Use procps for memory profiling"
  ON
)

option(
  VERBOSE
  "Print internal messages"
  OFF
)

if(CMAKE_COMPILER_IS_GNUCXX OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  # Common compilation flags and warning configuration
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Wextra -Wfatal-errors -pthread")

  if("${MULTICORE}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fopenmp")
  endif()

   # Default optimizations flags (to override, use -DOPT_FLAGS=...)
  if("${OPT_FLAGS}" STREQUAL "")
    set(OPT_FLAGS "-ggdb3 -O2 -march=native -mtune=native")
  endif()
endif()

add_definitions(-DCURVE_${CURVE})

if(${CURVE} STREQUAL "BN128")
  add_definitions(-DBN_SUPPORT_SNARK=1)
endif()

if("${VERBOSE}")
  add_definitions(-DVERBOSE=1)
endif()

if("${MULTICORE}")
  add_definitions(-DMULTICORE=1)
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OPT_FLAGS}")

include(FindPkgConfig)
if("${WITH_PROCPS}")
  pkg_check_modules(PROCPS REQUIRED libprocps)
else()
  add_definitions(-DNO_PROCPS)
endif()

include_directories(.)

add_subdirectory(depends)
add_subdirectory(src)
```

Next, create a `CMakeLists.txt` file in the `depends` directory and include the `libsnark` dependency.
```
add_subdirectory(libsnark)
```

Lastly, create a `CMakeLists.txt` file in the `src` directory.

### Development Framework

Start by creating a boilerplate `src/main.cpp` file.
```cpp
int main () {
    return 0;
}
```

Next, create a `CMakeLists.txt` file in the `src` directory and link the `main.cpp` file as follows.
```
include_directories(.)

add_executable(
  main

  main.cpp
)
target_link_libraries(
  main

  snark
)
target_include_directories(
  main

  PUBLIC
  ${DEPENDS_DIR}/libsnark
  ${DEPENDS_DIR}/libsnark/depends/libfqfft
)
```

## zkSNARK Application

__This library includes the completed zkSNARK application. If you wish to use the provided environment, you may proceed to [Compilation](#compilation).__

Our application will make use of the [Groth16](https://github.com/scipr-lab/libsnark/tree/master/libsnark/zk_proof_systems/ppzksnark/r1cs_gg_ppzksnark) zkSNARK protocol as provided by `libsnark`. The protocol is comprised of a setup phase, proving phase, and verification phase. The setup is responsible for constructing the public parameters that the prover and verifier use. The prover provides their public and private inputs, along with the public parameters, to construct a succinct proof. Any verifier is then able to verify this proof using the verification key, public input, and proof. Note that the verification key is derived from the public parameters.

```cpp
template<typename ppT>
bool run_r1cs_gg_ppzksnark(const r1cs_example<libff::Fr<ppT> > &example)
{
    libff::print_header("R1CS GG-ppzkSNARK Generator");
    r1cs_gg_ppzksnark_keypair<ppT> keypair = r1cs_gg_ppzksnark_generator<ppT>(example.constraint_system);
    printf("\n"); libff::print_indent(); libff::print_mem("after generator");

    libff::print_header("Preprocess verification key");
    r1cs_gg_ppzksnark_processed_verification_key<ppT> pvk = r1cs_gg_ppzksnark_verifier_process_vk<ppT>(keypair.vk);

    libff::print_header("R1CS GG-ppzkSNARK Prover");
    r1cs_gg_ppzksnark_proof<ppT> proof = r1cs_gg_ppzksnark_prover<ppT>(keypair.pk, example.primary_input, example.auxiliary_input);
    printf("\n"); libff::print_indent(); libff::print_mem("after prover");

    libff::print_header("R1CS GG-ppzkSNARK Verifier");
    const bool ans = r1cs_gg_ppzksnark_verifier_strong_IC<ppT>(keypair.vk, example.primary_input, proof);
    printf("\n"); libff::print_indent(); libff::print_mem("after verifier");
    printf("* The verification result is: %s\n", (ans ? "PASS" : "FAIL"));

    libff::print_header("R1CS GG-ppzkSNARK Online Verifier");
    const bool ans2 = r1cs_gg_ppzksnark_online_verifier_strong_IC<ppT>(pvk, example.primary_input, proof);
    assert(ans == ans2);

    return ans;
}
```

Our application will construct a sample circuit, making use of the [binary input circuit](https://github.com/scipr-lab/libsnark/blob/master/libsnark/relations/constraint_satisfaction_problems/r1cs/examples/r1cs_examples.tcc#L100) example provided by `libsnark`.
```cpp
template<typename ppT>
void test_r1cs_gg_ppzksnark(size_t num_constraints, size_t input_size)
{
    r1cs_example<libff::Fr<ppT> > example = generate_r1cs_example_with_binary_input<libff::Fr<ppT> >(num_constraints, input_size);
    const bool bit = run_r1cs_gg_ppzksnark<ppT>(example);
    assert(bit);
}
```

Lastly, we invoke the methods and construct a circuit with 1000 constraints and an input size of 100 elements.
```cpp
int main () {
    default_r1cs_gg_ppzksnark_pp::init_public_params();
    test_r1cs_gg_ppzksnark<default_r1cs_gg_ppzksnark_pp>(1000, 100);

    return 0;
}
```

## Compilation

To compile this library, start by recursively fetching the dependencies.
```
git submodule update --init --recursive
```

Note, the submodules only need to be fetched once.

Next, initialize the `build` directory.
```
mkdir build && cd build && cmake ..
```

Lastly, compile the library.
```
make
```

To run the application, use the following command from the `build` directory:
```
./src/main
```

## Further Resources

### Libraries
* [libsnark](http://github.com/SCIPR-Lab/libsnark) - C++ library for zkSNARK proofs
* [libfqfft](https://github.com/scipr-lab/libfqfft) - C++ library for FFTs in Finite Fields
* [libff](https://github.com/scipr-lab/libff) - C++ library for Finite Fields and Elliptic Curves
* [Zcash](https://github.com/zcash/zcash) - Internet Money, an implementation of the Zerocash protocol
* [ZSL on Quorum](https://github.com/jpmorganchase/zsl-q) - Zero-knowledge security layer in JP Morgan Quorum
* [ZoKrates](https://github.com/JacobEberhardt/ZoKrates) - A toolbox for zkSNARKs on Ethereum

### Articles
* [zkSNARKs Under the Hood](https://medium.com/@VitalikButerin/zk-snarks-under-the-hood-b33151a013f6) - Vitalik Buterin
* [zkSNARKs in a nutshell](https://blog.ethereum.org/2016/12/05/zksnarks-in-a-nutshell/) - Christian Reitwiessner
* [What are zkSNARKs?](https://z.cash/technology/zksnarks.html) - Zcash
* [zkSNARK Reading List](https://tahoe-lafs.org/trac/tahoe-lafs/wiki/SNARKs) - Tahoe-LAFS

### Talks
* [Zerocash: Solving Bitcoin's Privacy Problem](https://www.youtube.com/watch?v=84Vbj7-i9CI) - Alessandro Chiesa
* [SNARKs and their Practical Applications](https://simons.berkeley.edu/talks/eran-tromer-2015-06-10) - Eran Tromer
* [Zcash, SNARKs, STARKs](https://www.youtube.com/watch?v=VUN35BC11Qw) - Eli Ben Sasson
* [Democratizing zkSNARKs](https://www.youtube.com/watch?v=7BxoyEw6LUY) - Sean Bowe

## License

[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

