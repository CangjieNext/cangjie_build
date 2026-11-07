# 提交信息本地校验指南（pre-commit）

## 1. 概述

仓颉社区各代码仓通过仓库根目录的 `.pre-commit-config.yaml` 统一配置提交信息校验规则。开发者完成本地环境配置后，`git commit` 时将自动执行与流水线门禁一致的检查，可在推送前发现并修正提交信息问题，避免因提交信息不合规导致流水线反复失败。

本文档面向代码贡献者，说明本地安装与使用方式。

## 2. 环境要求

- Git
- Python 3.9 及以上
- Node.js 与 npm（commitlint 依赖），二者需位于 `PATH` 中

版本确认命令：

```bash
git --version
python3 --version
node --version
npm --version
```

## 3. 安装 pre-commit

每台开发机安装一次即可。推荐使用 pipx，以避免影响系统或项目 Python 环境：

```bash
pipx install pre-commit
```

未安装 pipx 时，可使用 pip 安装至用户目录：

```bash
python3 -m pip install --user pre-commit
```

安装完成后确认版本：

```bash
pre-commit --version
```

如需配置 pip 镜像源，可在 `~/.config/pip/pip.conf` 中添加：

```ini
[global]
index-url = https://mirrors.huaweicloud.com/repository/pypi/simple
trusted-host = mirrors.huaweicloud.com
```

## 4. 启用仓库钩子

进入已克隆的代码仓根目录执行：

```bash
cd <repository>
pre-commit install --install-hooks
```

说明：

- `--install-hooks` 表示立即创建钩子运行环境；省略该参数时，环境将在首次提交时创建。
- 该操作仅对当前仓库生效，不会修改全局 Git 配置。
- 钩子安装于 `.git/hooks/` 目录，不会纳入版本管理。

仓库根目录已提供 `.pre-commit-config.yaml`，开发者无需自行创建或修改。

## 5. 提交信息规范

`.pre-commit-config.yaml` 中定义了两项检查：

| 检查项 | 校验内容 |
|---|---|
| `cangjie-commitlint` | 提交信息符合 Conventional Commits 规范 |
| `cangjie-dco` | 提交信息包含 `Signed-off-by` 签名行 |

提交信息格式：

```
<type>(<scope>): <subject>

<body>

Signed-off-by: Name <email@example.com>
```

- `type` 允许值：`feat`、`fix`、`docs`、`style`、`refactor`、`test`、`chore`、`revert`、`perf`、`build`、`ci`
- `scope` 为可选项
- 标题长度不超过 100 字符，`subject` 结尾不使用句号
- 正文每行不超过 100 字符

推荐使用 `-s` 参数提交，自动生成 DCO 签名行：

```bash
git commit -sm "feat: support multi-line string literal"
```

## 6. 校验失败处理

### 6.1 修改最近一次提交

提交信息格式不合规时，校验输出示例如下：

```
type must be one of [build, chore, ci, docs, feat, fix, perf, refactor, revert, style, test] [type-enum]
```

尚未推送时，直接修改提交信息：

```bash
git commit --amend
```

缺少 `Signed-off-by` 时，追加签名：

```bash
git commit --amend -s
```

### 6.2 修改多条历史提交

流水线门禁检查合并请求中的全部提交。若存在多条不合规提交，需修改提交历史：

```bash
git rebase -i origin/main
```

在编辑器中将要修改的提交对应操作由 `pick` 改为 `reword`，保存后依次编辑提交信息。rebase 过程同样会触发校验，需修改至校验通过。

修改完成后更新远端分支：

```bash
git push --force-with-lease
```

执行强制推送前，应确认该分支无其他协作者提交。

## 7. 校验结果确认

```bash
# 确认 commit-msg 钩子已安装
ls .git/hooks/commit-msg

# 使用示例提交信息手动校验
echo "feat: test message" > /tmp/msg.txt
pre-commit run cangjie-commitlint --hook-stage commit-msg --commit-msg-filename /tmp/msg.txt
```

## 8. 常见问题

| 现象 | 原因 | 处理方式 |
|---|---|---|
| `No .pre-commit-config.yaml file was found` | 当前目录非仓库根目录，或仓库未提供配置文件 | 在仓库根目录执行；若仓库确无该文件，说明尚未接入校验 |
| 停留在 `Installing environment for ...` | 正在下载 commitlint 依赖，网络访问较慢 | 配置 npm 镜像后重新执行：`npm config set registry https://repo.huaweicloud.com/repository/npm/` |
| 长时间停留在 `Installing environment`，日志显示下载 Node | 本机 Node 由 nvm 等版本管理工具安装于用户目录，pre-commit 无法识别 | 需仓库配置包含 `default_language_version: {node: system}`，请联系仓库维护者确认 |
| 首次提交耗时较长 | 钩子运行环境尚未创建 | 提前执行 `pre-commit install --install-hooks` |
| 提交时未触发任何校验 | 安装的钩子类型不正确 | 在仓库根目录重新执行 `pre-commit install`，确认输出包含 `commit-msg` |
| Windows 环境安装失败 | 仓库配置中的 `node: system` 在 Windows 上不受支持 | 建议使用 WSL 或 Linux/macOS 环境进行开发 |

## 9. 临时跳过校验

以下方式仅建议在特殊情况下使用：

```bash
# 跳过指定检查项
SKIP=cangjie-commitlint git commit -sm "feat: xxx"

# 跳过全部本地钩子
git commit --no-verify -sm "feat: xxx"
```

本地跳过不会改变门禁结果，推送后仍会被流水线拦截。

## 10. 卸载

```bash
cd <repository>
pre-commit uninstall
```

卸载仅影响当前仓库，不修改全局 Git 配置。
