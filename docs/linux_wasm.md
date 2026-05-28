# 仓颉 SDK Linux wasm 构建指导书

## 1 概述

本文档说明如何在 Linux 主机构建一套可运行的 Cangjie SDK，并补齐 `wasm32-unknown-unknown` 目标所需的 compiler、runtime 与 stdlib 产物。

wasm 构建链路与现有交叉编译目标保持一致：

1. 先构建 Linux 主机侧 `cjc`
2. 再构建 `wasm32` 目标侧产物
3. 最后用 `cjc --target wasm32-unknown-unknown` 做最小编译验证

## 2 前提条件

### 2.1 系统依赖

请先按 [Linux Native 构建指导](linux.md) 完成主机环境准备。

此外，wasm 目标默认依赖系统已安装好的 LLVM 15 与 WASI 组件。要求以下工具可直接从当前环境找到：

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

### 2.2 环境变量

#### (1) OpenSSL 环境变量设置

构建 Linux 主机侧 stdlib 仍然依赖 OpenSSL 3+，因此需要先设置 `OPENSSL_PATH`：

- 定位 OpenSSL 的 lib 目录

  库文件通常位于 `/usr/lib/x86_64-linux-gnu/`（x86_64）或 `/usr/lib/aarch64-linux-gnu/`（aarch64），可先确认：

  ```bash
  ls /usr/lib/x86_64-linux-gnu/libssl.so*
  ls /usr/lib/x86_64-linux-gnu/libcrypto.so*
  ```

- 设置 `OPENSSL_PATH`

  ```bash
  export OPENSSL_PATH=/path/to/openssl-3.x
  export LD_LIBRARY_PATH=$OPENSSL_PATH:$LD_LIBRARY_PATH
  ```

#### (2) 其他环境变量设置

在后续步骤开始前，建议统一设置以下环境变量：

```bash
export WORKSPACE=/path/to/workspace
export CANGJIE_VERSION=1.0.0
export BUILD_TYPE=release
```

开发调试时，可将 `BUILD_TYPE` 改为 `relwithdebinfo`。

wasm 目标默认直接使用系统环境中的 LLVM 15 与 WASI 组件；只要 `clang`、`llvm-config`、`wasm-ld` 等工具可直接从当前 shell 找到，就不需要额外设置 `WASM_SYSROOT`、`--target-sysroot` 或版本化 `-B` 参数。

### 2.3 源码布局

建议工作区布局如下：

```text
$WORKSPACE
├── cangjie_build
├── cangjie_compiler
├── cangjie_runtime
└── llvm-project
```

构建 compiler 前，请先将自维护的 `llvm-project` 放入 `cangjie_compiler/third_party/llvm-project`：

```bash
cd $WORKSPACE/cangjie_compiler/third_party
git clone /path/to/llvm-project llvm-project
```

### 2.4 主机架构变量

后续复制 runtime 产物时需要用到主机架构目录名：

```bash
HOST_ARCH=$(uname -m)
if [ "${HOST_ARCH}" = "arm64" ]; then
  HOST_ARCH=aarch64
elif [ "${HOST_ARCH}" = "amd64" ]; then
  HOST_ARCH=x86_64
fi
```

## 3 构建流程

### 3.1 构建 Linux 主机侧 compiler

```bash
cd $WORKSPACE/cangjie_compiler
python3 build.py clean
python3 build.py build -t ${BUILD_TYPE} \
  -v ${CANGJIE_VERSION} \
  --product cjc \
  --no-tests
python3 build.py install
```

加载主机 SDK 环境：

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc -v
```

### 3.2 构建 Linux 主机侧 runtime

```bash
cd $WORKSPACE/cangjie_runtime/runtime
python3 build.py clean
python3 build.py build -t ${BUILD_TYPE} -v ${CANGJIE_VERSION}
python3 build.py install
cp -R output/common/linux_${BUILD_TYPE}_${HOST_ARCH}/{lib,runtime} \
  $WORKSPACE/cangjie_compiler/output
```

### 3.3 构建 Linux 主机侧 stdlib

```bash
cd $WORKSPACE/cangjie_runtime/stdlib
python3 build.py clean
python3 build.py build -t ${BUILD_TYPE} \
  --target-lib=$WORKSPACE/cangjie_runtime/runtime/output \
  --target-lib=$OPENSSL_PATH
python3 build.py install
cp -R output/* $WORKSPACE/cangjie_compiler/output/
```

## 4 构建 wasm 目标侧产物

### 4.1 构建 wasm compiler 依赖库

```bash
cd $WORKSPACE/cangjie_compiler
python3 build.py build -t ${BUILD_TYPE} \
  --product libs \
  --target wasm32 \
  --no-tests
python3 build.py install
```

### 4.2 构建 wasm runtime

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

### 4.3 构建 wasm stdlib

```bash
cd $WORKSPACE/cangjie_runtime/stdlib
python3 build.py clean
python3 build.py build -t ${BUILD_TYPE} \
  --target wasm32 \
  --target-lib=$WORKSPACE/cangjie_runtime/runtime/output
python3 build.py install
cp -R output/* $WORKSPACE/cangjie_compiler/output/
```

## 5 最小编译验证

准备样例 `add.cj`：

```cangjie
public func add(a: Int64, b: Int64): Int64 {
    a + b
}
```

执行：

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --experimental \
  --target wasm32-unknown-unknown \
  --output-type=obj \
  --compile-target=staticlib \
  add.cj \
  -o add.wasm.o
```

如需保留中间产物：

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
