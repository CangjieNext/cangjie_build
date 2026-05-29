# 仓颉 SDK Linux wasm 构建指导书

## 1 概述

本文档说明如何在 Linux 主机上构建带 wasm 目标支持的仓颉 SDK，并生成 `wasm32` 目标所需的 compiler、runtime 与 stdlib 产物。

当前 wasm 目标仍属于实验能力，编译 wasm 产物时需要显式使用 `--experimental`。

wasm 目标的用户可见写法为 `--target wasm32`。

## 2 前提条件

### 2.1 系统依赖

请先按 [Linux Native 构建指导](linux.md) 完成主机环境准备。

此外，wasm 目标需要 LLVM 15 与 WASI 相关组件。请确认以下工具可直接从当前 shell 找到：

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

构建 Linux 主机侧 stdlib 仍然依赖 OpenSSL 3+，因此需要先设置 `OPENSSL_PATH`：

```bash
export OPENSSL_PATH=/path/to/openssl-3.x
export LD_LIBRARY_PATH=$OPENSSL_PATH:$LD_LIBRARY_PATH
```

设置通用构建变量：

```bash
export WORKSPACE=/path/to/workspace
export CANGJIE_VERSION=1.0.0
export BUILD_TYPE=release
```

开发调试时，可将 `BUILD_TYPE` 改为 `relwithdebinfo`。

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

## 5 最小运行验证

### 5.1 生成 wasm 可执行产物

准备样例 `hello.cj`：

```cangjie
main() {
    println("hello cangjie")
}
```

编译 `wasm` 可执行文件：

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --experimental \
  --target wasm32 \
  --output-type=exe \
  hello.cj \
  -o hello.wasm
```

### 5.2 使用 Node.js + WASI 运行 wasm 文件

仓颉 wasm `exe` 产物是 WASI command module，可使用 Node.js + WASI 做最小运行验证。

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

运行：

```bash
node --no-warnings run-wasi.js hello.wasm
```

成功时应看到：

```text
hello cangjie
```

`--no-warnings` 用于关闭 Node.js WASI API 的 `ExperimentalWarning`。

### 5.3 wasm 动态库模块产物

如果目标是生成一个不包含 `main()` 的 `.wasm` 模块，并由 JavaScript 直接实例化后调用导出函数，则使用 `--output-type=dylib`。

准备样例 `add.cj`：

```cangjie
public func add(a: Int32, b: Int32): Int32 {
    return a + b
}
```

编译 `.wasm` 模块：

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --experimental \
  --target wasm32 \
  --output-type=dylib \
  add.cj \
  -o add.wasm
```

准备 `call-add.js`：

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

运行：

```bash
node --no-warnings call-add.js add.wasm
```

成功时应看到：

```text
5
```

当前阶段，JavaScript 示例优先查找用户可见导出名 `add`，并兼容当前编译器生成的仓颉符号名。稳定的 JavaScript 导出名机制仍在完善中。

### 5.4 wasm 静态库产物

如果目标不是“直接运行”，而是“给后续链接阶段复用”，则使用 `--output-type=staticlib`，产物是 `.a`：

```bash
source $WORKSPACE/cangjie_compiler/output/envsetup.sh
cjc --experimental \
  --target wasm32 \
  --output-type=staticlib \
  add.cj \
  -o libadd.a
```

这里的 `.a` 是 wasm 静态库归档，内部成员是 wasm object，用途和 native 静态库类似，但它本身不是可执行程序。
