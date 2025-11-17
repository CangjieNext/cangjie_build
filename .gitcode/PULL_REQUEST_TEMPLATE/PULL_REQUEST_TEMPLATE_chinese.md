## 变更摘要（必填）

请简要说明本 PR 对仓颉 SDK 集成构建指导文档（本仓库文档）的修改内容。

## 影响的文档（必填）

请勾选本次修改涉及的文件，编辑时将 `[ ]` 改为 `[x]`：

- [ ] README_zh.md
- [ ] docs/env_zh.md（工具链/版本矩阵）
- [ ] docs/linux_zh.md（Linux 原生构建）
- [ ] docs/macos_zh.md（macOS 原生构建）
- [ ] docs/linux_cross_windows_zh.md（Linux -> Windows 交叉构建）
- [ ] docs/linux_ohos_zh.md（Linux -> OHOS 交叉构建）
- [ ] docs/linux_cross_windows_ohos_zh.md（Windows -> OHOS，通过 Linux）
- [ ] docs/macos_ohos_zh.md（macOS -> OHOS 交叉构建）
- [ ] docs/linux_ohos_toolchain.md
- [ ] 其他：________

## 变更类型（必填）

- [ ] 新增文档/章节
- [ ] 更新现有步骤/命令
- [ ] 纠正错误的版本/工具名称
- [ ] 结构重组/表述更清晰
- [ ] 仅排版/错别字
- [ ] 废弃或标记过时内容

## 背景与动机

为什么需要这次修改？请指出具体问题、过时步骤或新的工具链要求。

## 验证证明（当修改命令或步骤时必填）

请提供可证明文档步骤可复现的证据。

- 宿主操作系统与版本：（例如 Ubuntu 22.04 / macOS 14）
- 目标平台：（例如 Linux / Windows / OHOS）
- CPU 架构：（例如 x86_64、arm64）
- 实际使用的工具版本（需与 docs/env_zh.md 的矩阵一致）：
  - python：__
  - clang：__
  - cmake：__
  - ninja：__
  - binutils/gcc 或 Xcode/clang（按需）：__
  - openssl/m4/mingw-w64（如适用）：__
- 实际执行的命令（从指南中复制，并补全变量）：

```
# 在此粘贴命令
```

- 日志/截图，应包含：
  - 配置/构建成功的关键信息
  - 若文档期望有运行结果，也请附上

> 说明：凡涉及命令或环境要求的修改，都应附带日志；否则可能被要求补充证据或标记为需要更多信息。

## 一致性检查（必填）

请勿删除检查项，完成后将 `[ ]` 改为 `[x]`。

- [ ] 版本与范围严格遵循 docs/env_zh.md 对应操作系统的矩阵。
- [ ] 文档间交叉链接已校对并更新（例如 README_zh.md 指向的链接正确）。
- [ ] Shell 命令可按文档原样复现（无缺失变量/路径）。
- [ ] 代码块使用正确的 shell 语言与一致的格式。
- [ ] 所有链接均可访问。
- [ ] 术语与本仓库保持一致（如 “Linux -> Windows 交叉构建”）。
- [ ] 若废弃内容，已在相关文档处标注替代方案并更新引用。
- [ ] 不涉及（本次未修改命令/版本）。

## 影响评估

- [ ] 可能需要后续更新 CI/示例
- [ ] 对读者无破坏性影响

## 关联的 Issue（必填）

请填写与本次 PR 相关的 issue 链接。