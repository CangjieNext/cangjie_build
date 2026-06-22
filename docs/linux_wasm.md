# 仓颉 SDK Linux WebAssembly 构建指导书

## 1 概述

本文档说明如何在 Linux 主机上构建带 WebAssembly 目标支持的仓颉 SDK，并使用该 SDK 生成和运行 wasm 产物。

仓颉 WebAssembly 提供两个用户目标：

| Target | Profile | 主要用途 |
| --- | --- | --- |
| `wasm32-wasi` | WASI profile | 构建 WASI command module、WASI reactor module 和 wasm archive。 |
| `wasm32-js` | JavaScript host profile | 构建 JavaScript runnable app、JavaScript host module 和 wasm archive。 |

SDK 支持以下 wasm 产物：

| Target | 产物类型 | 编译选项 | 产物 | 用途 |
| --- | --- | --- | --- | --- |
| `wasm32-wasi` | WASI command module | `--output-type=exe` | `<name>.wasm` | 运行包含 `main` 的命令行程序。 |
| `wasm32-wasi` | WASI reactor module | `--output-type=dylib` | `<name>.wasm` | 由 WASI host 实例化并初始化。 |
| `wasm32-js` | JavaScript runnable app | `--output-type=exe` | `<name>.wasm`、`<name>.mjs` | 由 JavaScript adapter 启动 `main`。 |
| `wasm32-js` | JavaScript host module | `--output-type=dylib` | `<name>.wasm`、`<name>.mjs`、`<name>.d.ts` | 由 JavaScript adapter 实例化并调用导出函数。 |
| `wasm32-wasi` / `wasm32-js` | wasm archive | `--output-type=staticlib` | `lib<name>.a` | 作为后续 wasm final link 的链接期依赖。 |

`cjpm` 的 `dynamic` product 使用 `dylib` 编译选项生成 wasm module 产物。

## 2 前提条件

### 2.1 系统工具安装

请先按 [Linux Native 构建指导](linux.md) 完成主机环境准备。

在 Linux Native 构建依赖基础上，wasm 目标还需要安装以下系统工具和 WASI 目标库：

```bash
sudo apt update && sudo apt install -y \
  llvm-15 llvm-15-dev lld-15 \
  wasi-libc libc++-15-dev-wasm32 libc++abi-15-dev-wasm32 \
  libclang-rt-15-dev-wasm32 nodejs
```

### 2.2 设置环境变量

#### (1) OpenSSL 环境变量设置

Linux 主机侧 stdlib/stdx 构建依赖 OpenSSL 3+。请沿用 [Linux Native 构建指导](linux.md) 中的 OpenSSL 路径设置：

```bash
export OPENSSL_PATH=/path/to/openssl-3.x
export LD_LIBRARY_PATH=$OPENSSL_PATH:$LD_LIBRARY_PATH
```

#### (2) 其他环境变量设置

在进行后续操作步骤前，需要配置以下环境变量：

```bash
export PATH=/usr/lib/llvm-15/bin:$PATH; # 用于将 clang-15 和 LLVM 15 工具加入环境变量
export WORKSPACE=/path/to/workspace
export ARCH=x86_64 # 或 aarch64
export CANGJIE_VERSION=1.0.0
export BUILD_TYPE=release
```

调试构建可使用：

```bash
export BUILD_TYPE=relwithdebinfo
```

设置 wasm target 与 SDK layout 变量：

```bash
export WASM_WASI_TARGET=wasm32-wasi
export WASM_JS_TARGET=wasm32-js
export WASM_WASI_LAYOUT=wasm32_wasi_cjnative
export WASM_JS_LAYOUT=wasm32_js_cjnative
```

环境变量设置完成后，可执行以下命令确认工具可从当前 shell 找到：

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
node --version
```

### 2.3 源码布局

推荐工作区布局：

```text
$WORKSPACE
├── cangjie_build
├── cangjie_compiler
├── cangjie_runtime
├── cangjie_stdx
├── cangjie_tools
└── llvm-project
```

构建 compiler 前，将自维护的 `llvm-project` 放入 compiler third_party 目录：

```bash
cd $WORKSPACE/cangjie_compiler/third_party
git clone /path/to/llvm-project llvm-project
```

## 3 构建 Linux 主机侧 SDK

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
cp -R output/common/linux_${BUILD_TYPE}_${ARCH}/{lib,runtime} \
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
for TARGET in ${WASM_WASI_TARGET} ${WASM_JS_TARGET}; do
  python3 build.py build -t ${BUILD_TYPE} \
    --product libs \
    --target ${TARGET} \
    --no-tests
  python3 build.py install --host ${TARGET}
  cp -R output-${TARGET}/* $WORKSPACE/cangjie_compiler/output/
done
```

### 4.2 构建 wasm runtime

```bash
cd $WORKSPACE/cangjie_runtime/runtime
python3 build.py clean
for TARGET in ${WASM_WASI_TARGET} ${WASM_JS_TARGET}; do
  python3 build.py build -t ${BUILD_TYPE} \
    --target ${TARGET} \
    -v ${CANGJIE_VERSION}
  python3 build.py install
done
cp -R output/common/wasm_${BUILD_TYPE}_wasm32/{lib,include} \
  $WORKSPACE/cangjie_compiler/output
```

### 4.3 构建 wasm stdlib

```bash
cd $WORKSPACE/cangjie_runtime/stdlib
python3 build.py clean
for TARGET in ${WASM_WASI_TARGET} ${WASM_JS_TARGET}; do
  python3 build.py build -t ${BUILD_TYPE} \
    --target ${TARGET} \
    --target-lib=$WORKSPACE/cangjie_runtime/runtime/output
  python3 build.py install --host ${TARGET}
  cp -R output-${TARGET}/* $WORKSPACE/cangjie_compiler/output/
done
```

### 4.4 构建 wasm stdx

```bash
cd $WORKSPACE/cangjie_stdx
python3 build.py clean
for TARGET in ${WASM_WASI_TARGET} ${WASM_JS_TARGET}; do
  python3 build.py build -t ${BUILD_TYPE} \
    --target ${TARGET} \
    --target-lib=$WORKSPACE/cangjie_runtime/runtime/output \
    --target-lib=$WORKSPACE/cangjie_runtime/stdlib/output
  python3 build.py install --host ${TARGET}
  cp -R output-${TARGET}/* $WORKSPACE/cangjie_compiler/output/
done
```

## 5 SDK 产物布局

完成构建后，SDK 输出目录包含：

```text
$WORKSPACE/cangjie_compiler/output
├── bin/cjc
├── envsetup.sh
├── lib/wasm32_wasi_cjnative/
├── runtime/lib/wasm32_wasi_cjnative/
├── modules/wasm32_wasi_cjnative/
├── lib/wasm32_js_cjnative/
├── runtime/lib/wasm32_js_cjnative/
├── modules/wasm32_js_cjnative/
├── templates/wasm/
└── third_party/llvm/bin/
    ├── opt
    ├── llc
    ├── wasm-ld
    └── llvm-ar
```

目录说明：

| 路径 | 内容 |
| --- | --- |
| `lib/wasm32_wasi_cjnative/` | `wasm32-wasi` stdlib archive、startup object、boundscheck archive。 |
| `runtime/lib/wasm32_wasi_cjnative/` | `wasm32-wasi` runtime archive。 |
| `modules/wasm32_wasi_cjnative/` | `wasm32-wasi` 目标 `.cjo` 包元数据。 |
| `lib/wasm32_js_cjnative/` | `wasm32-js` stdlib archive、boundscheck archive。 |
| `runtime/lib/wasm32_js_cjnative/` | `wasm32-js` runtime archive。 |
| `modules/wasm32_js_cjnative/` | `wasm32-js` 目标 `.cjo` 包元数据。 |
| `templates/wasm/` | JavaScript adapter 的 `.mjs`、`.d.ts` 模板。 |
| `third_party/llvm/bin/` | wasm codegen 和 linker 工具。 |

## 6 运行 `wasm32-wasi` command module

### 6.1 编写样例

创建 `hello.cj`：

```cangjie
main() {
    println("hello cangjie")
}
```

### 6.2 编译

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --target wasm32-wasi \
  --output-type=exe \
  hello.cj \
  -o hello.wasm
```

### 6.3 运行

准备 `run-wasi.js`：

```javascript
const fs = require('fs');
const { WASI } = require('wasi');

const wasmFile = process.argv[2];
const wasi = new WASI({
  version: 'preview1',
  args: [wasmFile],
  env: process.env,
  preopens: {},
  returnOnExit: true,
});

const env = new Proxy({}, {
  get(target, name) {
    if (name in target) return target[name];
    return (..._args) => (name === '__main_argc_argv' ? 0 : 0);
  },
});

async function main() {
  const bytes = fs.readFileSync(wasmFile);
  const { instance } = await WebAssembly.instantiate(bytes, {
    env,
    wasi_snapshot_preview1: wasi.wasiImport,
  });
  const exitCode = wasi.start(instance);
  if (typeof exitCode === 'number') {
    process.exit(exitCode);
  }
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

运行：

```bash
node --no-warnings run-wasi.js hello.wasm
```

输出：

```text
hello cangjie
```

## 7 运行 `wasm32-js` app 和 module

### 7.1 编写样例

创建 `hello.cj`：

```cangjie
main() {
    println("hello cangjie")
}
```

### 7.2 编译 JavaScript runnable app

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --target wasm32-js \
  --output-type=exe \
  hello.cj \
  -o hello.wasm
```

编译产物：

```text
hello.wasm
hello.mjs
```

运行：

```bash
node --no-warnings hello.mjs
```

输出：

```text
hello cangjie
```

### 7.3 编写导出函数

创建 `math_api.cj`：

```cangjie
@JSExport
public func add(a: Int32, b: Int32): Int32 {
    return a + b
}
```

### 7.4 编译 JavaScript host module

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --target wasm32-js \
  --output-type=dylib \
  math_api.cj \
  -o math_api.wasm
```

编译产物：

```text
math_api.wasm
math_api.mjs
math_api.d.ts
```

### 7.5 使用 JavaScript adapter

创建 `use_math_api.mjs`：

```javascript
import init, { add } from "./math_api.mjs";

await init({
  module: "math_api.wasm",
  args: ["math_api.wasm"],
});

console.log(add(2, 3));
```

运行：

```bash
node --no-warnings use_math_api.mjs
```

输出：

```text
5
```

## 8 构建并复用 wasm archive

### 8.1 构建链接期依赖

创建 `math_lib.cj`：

```cangjie
package mathlib

public func add(a: Int32, b: Int32): Int32 {
    return a + b
}
```

编译 `wasm32-wasi` archive：

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --target wasm32-wasi \
  --output-type=staticlib \
  -p mathlib \
  math_lib.cj \
  -o libmathlib.a
```

产物：

```text
libmathlib.a
mathlib.cjo
```

### 8.2 链接为 command module

创建 `main.cj`：

```cangjie
import mathlib.*

main() {
    println(add(20, 22))
}
```

编译最终 module：

```bash
cjc --target wasm32-wasi \
  main.cj \
  -L . \
  -l mathlib \
  --import-path . \
  -o main.wasm
```

运行：

```bash
node --no-warnings run-wasi.js main.wasm
```

输出：

```text
42
```

## 9 cjpm 构建

WASI profile 包项目使用：

```bash
cjpm build --target wasm32-wasi
```

JavaScript host profile 包项目使用：

```bash
cjpm build --target wasm32-js
```

`cjpm` 对依赖包生成 wasm archive。`wasm32-wasi` 对最终 executable product 生成 WASI command module，对最终 dynamic product 生成 WASI reactor module。`wasm32-js` 对最终 executable product 生成 JavaScript runnable app，对最终 dynamic product 生成 JavaScript host module。

## 10 Smoke 检查清单

构建完成后建议验证：

- `cjc --target wasm32-wasi --output-type=exe`
- `cjc --target wasm32-wasi --output-type=dylib`
- `cjc --target wasm32-js --output-type=exe`
- `cjc --target wasm32-js --output-type=dylib`
- `cjc --target wasm32-wasi --output-type=staticlib`
- `cjc --target wasm32-js --output-type=staticlib`
- `-L` / `-l` 链接 wasm archive
- Node/WASI 运行 WASI command module
- JavaScript adapter 调用 `@JSExport` 函数
- `cjpm build --target wasm32-wasi`
- `cjpm build --target wasm32-js`
- wasm-compatible stdx 包导入和链接
