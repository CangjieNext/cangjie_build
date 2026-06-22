# 仓颉 WebAssembly Build 规格

## 1. 范围

本文档定义集成构建仓颉 WebAssembly SDK 的构建顺序、安装布局、Linux 主机依赖、产物模型和 smoke 验证。

用户可见目标名为：

```text
wasm32-wasi
wasm32-js
```

SDK 安装布局使用：

```text
wasm32_wasi_cjnative
wasm32_js_cjnative
```

## 2. 构建顺序

集成构建按以下顺序产出 wasm SDK：

1. 构建带 WebAssembly 支持的 LLVM 工具。
2. 构建 Linux 主机侧 compiler。
3. 构建 Linux 主机侧 runtime 和 stdlib。
4. 构建 wasm compiler 依赖库。
5. 构建 `wasm32-wasi` runtime archive 和 startup object。
6. 构建 `wasm32-js` runtime archive。
7. 构建 `wasm32-wasi` stdlib archive 和 `.cjo` 包元数据。
8. 构建 `wasm32-js` stdlib archive 和 `.cjo` 包元数据。
9. 构建 wasm-compatible stdx 包。
10. 构建 `cjpm` 和工具链辅助工具。
11. 执行 smoke 验证。

后续步骤消费前序步骤安装到 SDK 输出目录的产物。

## 3. 安装布局

SDK 输出目录包含：

```text
bin/cjc
envsetup.sh
tools/bin/cjpm
lib/wasm32_wasi_cjnative/
runtime/lib/wasm32_wasi_cjnative/
modules/wasm32_wasi_cjnative/
lib/wasm32_js_cjnative/
runtime/lib/wasm32_js_cjnative/
modules/wasm32_js_cjnative/
templates/wasm/
third_party/llvm/bin/opt
third_party/llvm/bin/llc
third_party/llvm/bin/wasm-ld
third_party/llvm/bin/llvm-ar
```

`templates/wasm/` 提供 JavaScript adapter 模板。`modules/<layout>/` 提供 wasm 目标 `.cjo` 包元数据。`lib/<layout>/` 和 `runtime/lib/<layout>/` 提供 wasm archive 与 startup object。

## 4. wasm 产物

| Target | 产物类型 | 编译选项 | 产物 | 用途 |
| --- | --- | --- | --- | --- |
| `wasm32-wasi` | WASI command module | `--output-type=exe` | `<name>.wasm` | 由 WASI host 启动并运行 `main`。 |
| `wasm32-wasi` | WASI reactor module | `--output-type=dylib` | `<name>.wasm` | 由 WASI host 实例化并初始化。 |
| `wasm32-js` | JavaScript runnable app | `--output-type=exe` | `<name>.wasm`、`<name>.mjs` | 由 JavaScript adapter 启动 `main`。 |
| `wasm32-js` | JavaScript host module | `--output-type=dylib` | `<name>.wasm`、`<name>.mjs`、`<name>.d.ts` | 由 JavaScript adapter 实例化并调用导出函数。 |
| `wasm32-wasi` / `wasm32-js` | wasm archive | `--output-type=staticlib` | `lib<name>.a` | 作为后续 wasm final link 的链接期依赖。 |

`cjpm` 的 `dynamic` product 使用 `dylib` 编译选项生成 wasm module 产物。

## 5. Linux 构建输入

Linux 构建环境提供：

- Python 3
- CMake
- Ninja
- clang / clang++
- wasm-ld
- llvm-ar
- llvm-ranlib
- llvm-config
- WASI libc / libc++ / libc++abi / compiler-rt builtins
- OpenSSL 3+（用于 Linux 主机侧 stdlib/stdx 构建）

构建脚本从 SDK 输出目录和系统 LLVM/WASI 包安装路径解析 wasm 链接依赖。

## 6. 运行方式

`wasm32-wasi` 的 `exe` 产物使用 WASI host 运行：

```text
node --no-warnings run-wasi.js hello.wasm
```

`wasm32-wasi` 的 `dylib` 产物使用 WASI reactor runner 运行。

`wasm32-js` 的 `exe` 产物使用生成的 JavaScript app 运行：

```text
node --no-warnings hello.mjs
```

`wasm32-js` 的 `dylib` 产物使用生成的 JavaScript adapter 实例化 module、初始化 Cangjie runtime，并暴露 `@JSExport` 函数。

## 7. Smoke 验证

集成构建验证：

- `cjc --target wasm32-wasi --output-type=exe`
- `cjc --target wasm32-wasi --output-type=dylib`
- `cjc --target wasm32-js --output-type=exe`
- `cjc --target wasm32-js --output-type=dylib`
- `cjc --target wasm32-wasi --output-type=staticlib`
- `cjc --target wasm32-js --output-type=staticlib`
- archive 参与最终 `.wasm` 链接
- Node/WASI 运行 WASI command module
- Node 运行 JavaScript runnable app
- JavaScript adapter 调用 `@JSExport` 函数
- `cjpm build --target wasm32-wasi`
- `cjpm build --target wasm32-js`
- wasm-compatible stdx 包构建与导入
