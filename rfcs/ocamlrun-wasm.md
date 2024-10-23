# ocamlrun.wasm

## Introduction

### Context

The JsCoq authors (@ejgallego, @corwin-of-amber) maintain an in-browser version of Coq for teaching purposes ( https://coq.vercel.app/ ), produced using `js_of_ocaml` since 2015. This does not always work well, some parts of the Coq codebase (for example the `ring` tactic) fail with `Stack_overflow` due to partial tail-call support in `js_of_ocaml`. As an alternative, in 2019 @corwin-of-amber a WaCoq variant that relies on WebAssembly (wasm) instead of Javascript ( https://coq.vercel.app/wa/ ); it has comparable performance, and does not suffer from `Stack_overflow` issues.

The toolchain @corwin-of-amber uses is *not* the upcoming `wasm_of_ocaml` variant of `js_of_ocaml` ( https://github.com/ocaml-wasm/wasm_of_ocaml ), which compiles OCaml code into wasm. Instead, WaCoq compiles the ocaml bytecode interpreter `ocamlrun` (implemented in C) to wasm using the wasm LLVM backend distributed with Clang, and then uses the resulting `ocamlrun.wasm` (wasm) binary to execute (ocamlc and) Coq as bytecode programs.

### Proposal

This RFC proposes to make this low-tech wasm approach easy to use from the unmodified OCaml distribution. Currently @corwin-of-amber uses a patched version of 4.14 ( https://github.com/corwin-of-amber/ocaml-wasm/tree/wasm-4.14 ), and our proposal is to make it easy to build `ocamlrun.wasm` from the upstream OCaml distribution, to reduce the maintenance burden on @corwin-of-amber and make the approach available to other prospective users.

### Why not use `wasm_of_ocaml` instead?

It is interesting to contrast this "low-tech" approach to wasm distribution (using ocamlrun.wasm with stock OCaml bytecode programs) to the "native" approach offered by `wasm_of_ocaml`. We believe that both are valuable and make sense, for different audiences.

The low-tech approach has been available for a few years, and more importantly it provides pixel-perfect compatibility with the standard OCaml runtime (it *is* the standard OCaml runtime), including weak pointers, (bytecode) dynlink, effect handlers, etc.

In comparison, using these features with `wasm_of_ocaml` is uncharted territory, becauses it relies on experimental wasm features that are not available in all browsers. It is reasonable to expect that it will take a few more years before Web browsers have wide support for the wasm extensions that are required to compile all of the Coq codebase to wasm directly, or other large and complex OCaml codebases out there that could benefit from web distribution.

We see `ocamlrun.wasm` and `wasm_of_ocaml` as complementary approaches targeting difference audiences:
- `ocamlrun.wasm` provides easy deployment of large OCaml codebases to the web, without the tail-call limitations of `js_of_ocaml`.
- `wasm_of_ocaml` can be expected to produce faster code, but its browser support is more experimental, especially for large programs using complex OCaml runtime features.


## Proposed goals

Currently the [patched ocaml-wasm compiler](https://github.com/corwin-of-amber/ocaml-wasm/tree/wasm-4.14) contains the following pieces:

- a patch to the runtime to be able to compile `ocamlrun` as a wasm program (compilation to wasm relies on a `wasi` runtime that does not provide full POSIX compatibility, but it supports enough of it to build the OCaml runtime, except with (currently) `setjmp/longjmp` support which is avoided by patching `interp.c` to use trampolines instead), relying on support libraries implemented by @corwin-of-amber as the [wasi-kernel](https://github.com/corwin-of-amber/wasi-kernel/) project.

- (hacky) build scripts and pre-configured .h files to call `clang` to produce
  `ocamlrun.wasm`

- build information to build `otherlibs/unix/` and `otherlibs/str/` using clang-to-wasm,
  and code to stub out the `otherlibs/systhreads` library (not currently supported,
  but could be added with extra development work)

- packaging scripts to distribute `ocamlrun.wasm` on npm, as well as `.wasm` versions
  of non-stdlib `.cma` files

Some of this are prototype-quality patches, some is outside the scope of the upstream OCaml distribution.
We propose a different scope for upstreaming with maintenability and minimality in mind:

- configure logic for a `wasm32-unknown-emscripten` bytecode-only target that
  configures the runtime for a wasm environment

- conditional logic in the OCaml bytecode interpreter to be able
  to build against the `wasi` runtime (this is a small patch, the
  current iteration is
  [ocaml-wasm:b58ac23cab](https://github.com/corwin-of-amber/ocaml-wasm/commit/b58ac23cabff8b6fe86b7233aab0a171097714c1);
  it could be simplified further or possibly removed entirely by
  looking at setjmp/longjmp support in more recent `wasi` versions
  
- a `Makefile` target to build `ocamlrun.wasm`, which relies on
  extra system dependencies compared to the rest of the OCaml
  distribution: `clang` must be available (a recent-enough version
  that supports the LLVM backend), and some `wasi-*` support libraries
  must have been installed with `npm`.

- a `testsuite/Makefile` target that exercises the OCaml testsuite
  running under `ocamlrun.wasm`, itself interpreted by `node` (NodeJS,
  which serves as a wasm interpreter on development machines).

- a CI script to build this and run the testsuite, to check that it works
  (but currently we are *not* proposing to enable it on the upstream CI
  system; ideally it would be run less often than other CI checks, for
  example on a weekly basis)

In particular, all npm packaging logic would remain outside the core
distribution.
