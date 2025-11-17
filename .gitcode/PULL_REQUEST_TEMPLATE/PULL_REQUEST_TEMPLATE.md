## Summary (Required)

Briefly describe what this PR changes in the Cangjie SDK build guides and/or repo docs.

## Affected Documents (Required)

Mark all that apply. Edit by changing `[ ]` to `[x]`.

- [ ] README_zh.md
- [ ] docs/env_zh.md (toolchain/version matrix)
- [ ] docs/linux_zh.md (Linux native)
- [ ] docs/macos_zh.md (macOS native)
- [ ] docs/linux_cross_windows_zh.md (Linux -> Windows)
- [ ] docs/linux_ohos_zh.md (Linux -> OHOS)
- [ ] docs/linux_cross_windows_ohos_zh.md (Windows -> OHOS via Linux)
- [ ] docs/macos_ohos_zh.md (macOS -> OHOS)
- [ ] docs/linux_ohos_toolchain.md
- [ ] Other: ________

## Change Type (Required)

- [ ] New guide/content
- [ ] Update existing steps/commands
- [ ] Fix incorrect versions/tool names
- [ ] Restructure/clarify sections
- [ ] Typo/style/formatting only
- [ ] Deprecate/outdate obsolete content

## Context and Motivation

Why is this change needed? Reference specific problems, outdated steps, or new toolchain requirements.

## Validation Evidence (Required when changing commands/steps)

Provide concrete evidence that the documented steps work.

- Host OS and version: (e.g., Ubuntu 22.04 / macOS 14)
- Target platform(s): (e.g., Linux / Windows / OHOS)
- CPU architecture: (e.g., x86_64, arm64)
- Tool versions used (must align with docs/env_zh.md):
  - python: __
  - clang: __
  - cmake: __
  - ninja: __
  - binutils/gcc or Xcode/clang as applicable: __
  - openssl/m4/mingw-w64 (if applicable): __
- Exact commands executed (copy from the guide with any variables resolved):

```
# paste commands here
```

- Logs/screenshots showing:
  - successful configure/build steps
  - any runtime results if the guide expects them

> Note: Changes that alter example commands or env requirements should include logs. PRs without evidence may be asked to add proof or will be marked as needs-more-info.

## Consistency Checks (Required)

Do not delete items. Mark completed items with `[x]`.

- [ ] Versions and ranges strictly follow docs/env_zh.md matrices for the relevant OS.
- [ ] Cross-links between guides are correct and updated (e.g., README_zh.md links to the right doc).
- [ ] All shell commands are reproducible as written (no missing variables/paths).
- [ ] Code blocks use the correct shell language and consistent formatting.
- [ ] All links are valid and reachable.
- [ ] Terminology matches repository usage (e.g., “Linux -> Windows 交叉构建”).
- [ ] If deprecating content, noted alternatives and updated references across files.
- [ ] Not applicable (no commands/versions changed)

## Potential Impact

- [ ] This change may require updating CI/examples later
- [ ] No breaking changes to readers

## Related Issues (Required)

Link to related issues here.