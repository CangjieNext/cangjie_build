# Cangjie Build WebAssembly Specification

## 1. Scope

This specification defines the integrated build requirements for Cangjie
WebAssembly support. It covers repository build order, install layout, Linux
host prerequisites, and smoke validation.

The user-facing target name is `wasm32`.

## 2. Build Order

An integrated wasm toolchain build produces, in order:

1. LLVM tools with wasm support
2. Cangjie runtime wasm libraries
3. Cangjie compiler with wasm target support
4. Cangjie standard library wasm archives
5. `stdx` wasm-compatible packages
6. `cjpm` and other tools
7. tests and smoke artifacts

Later steps consume the installed outputs of earlier steps. In particular,
`cjpm` depends on the compiler, standard library, runtime, and `stdx` install
layout.

## 3. Install Layout

The integrated output tree must contain:

```text
bin/cjc
tools/bin/cjpm
lib/wasm32_cjnative/
runtime/lib/wasm32_cjnative/
modules/wasm32_cjnative/
templates/wasm/
third_party/llvm/bin/opt
third_party/llvm/bin/llc
third_party/llvm/bin/wasm-ld
third_party/llvm/bin/llvm-ar
```

The compiler uses `templates/wasm/` to generate JavaScript loaders and
TypeScript declarations.

## 4. Linux Build Inputs

The Linux build environment must provide wasm-capable LLVM tools and wasm
system libraries, including WASI libc/c++ libraries and compiler-rt builtins.

The build scripts may use toolchain-provided LLVM tools from the Cangjie output
tree. Hard-coded host-only library paths must not be used for wasm final links.

## 5. User Build Guide Contract

The wasm build guide must show a minimal working flow:

1. build and install the toolchain
2. compile a Cangjie `main` program that prints `hello cangjie`
3. compile a no-main wasm module when demonstrating JavaScript interoperation
4. use the generated `.mjs` loader and `.d.ts` declarations
5. run the generated module with Node.js and WASI

The guide must pass only the wasm file name to the JavaScript loader unless
additional options are intentionally demonstrated.

## 6. Smoke Validation

The integrated build must validate:

- `cjc --target wasm32` executable output
- `cjc --target wasm32 --output-type=dylib` no-main output
- generated `.mjs` and `.d.ts` sidecars for `@JSExport`
- Node.js invocation through the generated loader
- `cjpm build --target wasm32` for package projects
- pure-data `stdx` package use, such as JSON or markdown processing when
  available

## 7. Unsupported Capabilities

Build documentation must identify unsupported or host-dependent capabilities:

- native dynamic loading
- OS threads
- process spawning
- native network sockets
- unrestricted file system access

Unsupported behavior belongs in runtime/library diagnostics, not in ambiguous
build failures.
