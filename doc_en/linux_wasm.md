# Cangjie SDK Linux wasm Build Guide

## 1 Overview

This document describes how to build a Linux-hosted Cangjie SDK and produce the compiler, runtime, and standard library artifacts required by the `wasm32-unknown-unknown` target.

The wasm flow follows the same structure as the existing cross-compilation targets:

1. Build the Linux host `cjc`
2. Build target-side artifacts for `wasm32`
3. Validate the path with a minimal `cjc --target wasm32-unknown-unknown` compilation

## 2 Prerequisites

### 2.1 System Dependencies

First complete the host environment preparation in the [Linux Native Build Guide](linux.md).

In addition, the wasm target expects LLVM 15 and WASI components to be installed in the system environment already. The following tools must be available directly from the current shell:

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

#### (1) OpenSSL Environment Variable Setup

The Linux host stdlib build still depends on OpenSSL 3+, so `OPENSSL_PATH` must be configured first:

- Locate the OpenSSL `lib` directory

  The libraries are typically under `/usr/lib/x86_64-linux-gnu/` on x86_64 or `/usr/lib/aarch64-linux-gnu/` on aarch64. Verify with:

  ```bash
  ls /usr/lib/x86_64-linux-gnu/libssl.so*
  ls /usr/lib/x86_64-linux-gnu/libcrypto.so*
  ```

- Set `OPENSSL_PATH`

  ```bash
  export OPENSSL_PATH=/path/to/openssl-3.x
  export LD_LIBRARY_PATH=$OPENSSL_PATH:$LD_LIBRARY_PATH
  ```

#### (2) Other Environment Variable Setup

Before continuing, set the common build variables:

```bash
export WORKSPACE=/path/to/workspace
export CANGJIE_VERSION=1.0.0
export BUILD_TYPE=release
```

For development and debugging builds, set `BUILD_TYPE=relwithdebinfo`.

The wasm target uses the system-provided LLVM 15 and WASI components directly. As long as `clang`, `llvm-config`, and `wasm-ld` are available in the current shell, no extra `WASM_SYSROOT`, `--target-sysroot`, or version-specific `-B` argument is required.

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

## 3 Build the Host SDK

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

## 5 Minimal Compile Validation

Prepare `add.cj`:

```cangjie
public func add(a: Int64, b: Int64): Int64 {
    a + b
}
```

Run:

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --experimental \
  --target wasm32-unknown-unknown \
  --output-type=obj \
  --compile-target=staticlib \
  add.cj \
  -o add.wasm.o
```

To keep temporary files:

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --experimental \
  --target wasm32-unknown-unknown \
  --output-type=obj \
  --compile-target=staticlib \
  add.cj \
  -o add.wasm.o \
  --save-temps wasm-temps
```
