# Cangjie SDK Linux wasm Build Guide

## 1 Overview

This document describes how to build a Linux-hosted Cangjie SDK with wasm target support and produce the compiler, runtime, and standard library artifacts required by the `wasm32` target.

The wasm target is currently experimental. Use `--experimental` when compiling wasm artifacts.

The user-facing wasm target spelling is `--target wasm32`.

## 2 Prerequisites

### 2.1 System Dependencies

First complete the host environment preparation in the [Linux Native Build Guide](linux.md).

In addition, the wasm target requires LLVM 15 and WASI-related components. The following tools must be available directly from the current shell:

```bash
python3 --version
cmake --version
ninja --version
clang --version
clang++ --version
wasm-ld --version
llvm-ar --version
llvm-ranlib --version
llvm-config --version
```

### 2.2 Environment Variables

The Linux host stdlib build still depends on OpenSSL 3+, so configure `OPENSSL_PATH` first:

```bash
export OPENSSL_PATH=/path/to/openssl-3.x
export LD_LIBRARY_PATH=$OPENSSL_PATH:$LD_LIBRARY_PATH
```

Set the common build variables:

```bash
export WORKSPACE=/path/to/workspace
export CANGJIE_VERSION=1.0.0
export BUILD_TYPE=release
```

For development and debugging builds, set `BUILD_TYPE=relwithdebinfo`.

### 2.3 Source Layout

Recommended workspace layout:

```text
$WORKSPACE
├── cangjie_build
├── cangjie_compiler
├── cangjie_runtime
└── llvm-project
```

Before building the compiler, place the maintained `llvm-project` tree under `cangjie_compiler/third_party/llvm-project`:

```bash
cd $WORKSPACE/cangjie_compiler/third_party
git clone /path/to/llvm-project llvm-project
```

### 2.4 Host Architecture Variable

The runtime artifact copy steps below use the normalized host architecture name:

```bash
HOST_ARCH=$(uname -m)
if [ "${HOST_ARCH}" = "arm64" ]; then
  HOST_ARCH=aarch64
elif [ "${HOST_ARCH}" = "amd64" ]; then
  HOST_ARCH=x86_64
fi
```

## 3 Build the Linux Host SDK

### 3.1 Build the Linux Host Compiler

```bash
cd $WORKSPACE/cangjie_compiler
python3 build.py clean
python3 build.py build -t ${BUILD_TYPE} \
  -v ${CANGJIE_VERSION} \
  --product cjc \
  --no-tests
python3 build.py install
```

Load the host SDK environment:

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc -v
```

### 3.2 Build the Linux Host Runtime

```bash
cd $WORKSPACE/cangjie_runtime/runtime
python3 build.py clean
python3 build.py build -t ${BUILD_TYPE} -v ${CANGJIE_VERSION}
python3 build.py install
cp -R output/common/linux_${BUILD_TYPE}_${HOST_ARCH}/{lib,runtime} \
  $WORKSPACE/cangjie_compiler/output
```

### 3.3 Build the Linux Host Standard Library

```bash
cd $WORKSPACE/cangjie_runtime/stdlib
python3 build.py clean
python3 build.py build -t ${BUILD_TYPE} \
  --target-lib=$WORKSPACE/cangjie_runtime/runtime/output \
  --target-lib=$OPENSSL_PATH
python3 build.py install
cp -R output/* $WORKSPACE/cangjie_compiler/output/
```

## 4 Build wasm Target Artifacts

### 4.1 Build wasm Compiler Support Libraries

```bash
cd $WORKSPACE/cangjie_compiler
python3 build.py build -t ${BUILD_TYPE} \
  --product libs \
  --target wasm32 \
  --no-tests
python3 build.py install
```

### 4.2 Build wasm Runtime

```bash
cd $WORKSPACE/cangjie_runtime/runtime
python3 build.py clean
python3 build.py build -t ${BUILD_TYPE} \
  --target wasm32 \
  -v ${CANGJIE_VERSION}
python3 build.py install
cp -R output/common/wasm_${BUILD_TYPE}_wasm32/{lib,include} \
  $WORKSPACE/cangjie_compiler/output
```

### 4.3 Build wasm Standard Library

```bash
cd $WORKSPACE/cangjie_runtime/stdlib
python3 build.py clean
python3 build.py build -t ${BUILD_TYPE} \
  --target wasm32 \
  --target-lib=$WORKSPACE/cangjie_runtime/runtime/output
python3 build.py install
cp -R output/* $WORKSPACE/cangjie_compiler/output/
```

## 5 Minimal Runtime Validation

### 5.1 Build the wasm Executable Artifact

Prepare `hello.cj`:

```cangjie
main() {
    println("hello cangjie")
}
```

Build the wasm executable:

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --experimental \
  --target wasm32 \
  --output-type=exe \
  hello.cj \
  -o hello.wasm
```

### 5.2 Run the wasm File with Node.js + WASI

Cangjie wasm `exe` output is a WASI command module. Use Node.js + WASI for minimal runtime validation.

Prepare `run-wasi.js`:

```javascript
const fs = require('fs');
const { WASI } = require('wasi');

const wasmFile = process.argv[2];
const wasi = new WASI({
  version: 'preview1',
  args: [wasmFile],
  env: process.env,
  preopens: {},
});

const env = new Proxy({}, {
  get(_target, name) {
    return (..._args) => {
      if (name === '__main_argc_argv') {
        return 0;
      }
      return 0;
    };
  },
});

async function main() {
  const bytes = fs.readFileSync(wasmFile);
  const { instance } = await WebAssembly.instantiate(bytes, {
    env,
    wasi_snapshot_preview1: wasi.wasiImport,
  });
  wasi.start(instance);
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

Run:

```bash
node --no-warnings run-wasi.js hello.wasm
```

On success, the output should include:

```text
hello cangjie
```

`--no-warnings` suppresses Node.js WASI `ExperimentalWarning` output.

### 5.3 wasm Dynamic Library Module Artifact

If the goal is to produce a `.wasm` module without `main()` and instantiate it directly from JavaScript, use `--output-type=dylib`.

Prepare `add.cj`:

```cangjie
public func add(a: Int32, b: Int32): Int32 {
    return a + b
}
```

Build the `.wasm` module:

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --experimental \
  --target wasm32 \
  --output-type=dylib \
  add.cj \
  -o add.wasm
```

Prepare `call-add.js`:

```javascript
const fs = require('fs');
const { WASI } = require('wasi');

const wasmFile = process.argv[2];
const wasi = new WASI({
  version: 'preview1',
  args: [wasmFile],
  env: process.env,
  preopens: {},
});

const env = new Proxy({}, {
  get(_target, name) {
    return (..._args) => {
      if (name === '__main_argc_argv') {
        return 0;
      }
      return 0;
    };
  },
});

function resolveAdd(exports) {
  if (typeof exports.add === 'function') {
    return exports.add;
  }
  const name = Object.keys(exports).find((key) => {
    return typeof exports[key] === 'function' && /3addHii$/.test(key);
  });
  return name ? exports[name] : undefined;
}

async function main() {
  const bytes = fs.readFileSync(wasmFile);
  const { instance } = await WebAssembly.instantiate(bytes, {
    env,
    wasi_snapshot_preview1: wasi.wasiImport,
  });
  const add = resolveAdd(instance.exports);
  if (typeof add !== 'function') {
    throw new Error('missing wasm export for add(Int32, Int32)');
  }
  console.log(add(2, 3));
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

Run:

```bash
node --no-warnings call-add.js add.wasm
```

On success, the output should include:

```text
5
```

For now, the JavaScript example first looks for the user-facing export name `add`, then falls back to the current Cangjie symbol name. A stable JavaScript export-name mechanism is still being improved.

### 5.4 wasm Static Library Artifact

If the goal is not to run the output directly, but to reuse it in a later link step, use `--output-type=staticlib`, which produces a `.a` archive:

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --experimental \
  --target wasm32 \
  --output-type=staticlib \
  add.cj \
  -o libadd.a
```

This `.a` file is a wasm static library archive whose members are wasm object files. It is link-time input, not a runnable program.
