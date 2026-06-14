# Repo 命令常用操作手册

本文档整理本工程中 `repo` 工具的常用操作，包括拉取代码、同步更新、查看状态、创建分支、提交代码和推送代码。命令默认在 SDK 根目录执行：

```bash
cd /home/hl/code/arm_sdk
```

## 1. 本工程 Repo 配置

manifest 文件：

```text
.repo/manifests/default.xml
```

当前默认配置：

| 配置项 | 值 |
| --- | --- |
| manifest 仓库 | `git@github.com:huangliang367/manifest.git` |
| remote | `git@github.com:huangliang367/` |
| 默认分支 | `main` |

当前包含的 project：

| Project | 本地路径 |
| --- | --- |
| `linux` | `linux/` |
| `u-boot` | `u-boot/` |
| `docs` | `docs/` |
| `scripts` | `scripts/` |
| `prebuild` | `prebuild/` |
| `dockerfile` | `dockerfile/` |

## 2. 安装 Repo

如果系统未安装 `repo`，可以使用顶层 Makefile：

```bash
make repo
```

也可以手动安装：

```bash
sudo curl -sSL -o /usr/local/bin/repo https://storage.googleapis.com/git-repo-downloads/repo
sudo chmod a+x /usr/local/bin/repo
```

检查：

```bash
repo version
```

## 3. 首次拉取代码

新建工作目录并初始化：

```bash
mkdir arm_sdk
cd arm_sdk
repo init -u git@github.com:huangliang367/manifest.git
repo sync
```

本工程顶层 `Makefile` 已封装该流程：

```bash
make manifest
```

`make manifest` 的行为：

- 如果 `.repo/` 不存在，执行 `repo init -u git@github.com:huangliang367/manifest.git`
- 然后执行 `repo sync`
- 如果 `.repo/` 已存在，跳过 init，直接 sync

## 4. 同步代码

同步所有 project：

```bash
repo sync
```

常用同步参数：

```bash
repo sync -j8
```

说明：

- `-j8` 表示 8 并发同步，可按机器和网络情况调整

只同步某个 project：

```bash
repo sync linux
repo sync u-boot
repo sync docs
```

丢弃本地未提交修改并强制同步指定 project：

```bash
repo sync --force-sync linux
```

注意：`--force-sync` 可能覆盖本地 checkout 状态。执行前先确认没有需要保留的本地修改：

```bash
repo status
```

如果只是普通更新，优先使用：

```bash
repo sync
```

## 5. 查看状态

查看所有 project 的修改状态：

```bash
repo status
```

进入某个 project 后，也可以使用普通 Git 命令：

```bash
cd docs
git status
```

查看每个 project 当前分支：

```bash
repo branches
```

查看 manifest 中的 project 列表：

```bash
repo list
```

只显示路径：

```bash
repo list -p
```

## 6. 创建开发分支

Repo 管理的是多个 Git 仓库。建议不要直接在 detached HEAD 上开发，先创建 topic branch。

在所有 project 上创建同名分支：

```bash
repo start my_feature --all
```

只在指定 project 上创建分支：

```bash
repo start my_feature docs
repo start my_feature linux
repo start my_feature u-boot
```

示例：只修改文档时，在 `docs` project 创建分支：

```bash
repo start update_repo_docs docs
cd docs
git status
```

如果已经进入某个 project，也可以用普通 Git：

```bash
cd docs
git checkout -b update_repo_docs
```

但更推荐使用 `repo start`，因为它会记录 topic branch，后续 `repo upload` 更方便识别。

## 7. 提交代码

`repo` 不替代 Git 提交。每个 project 仍然需要用 Git 单独提交。

示例：提交 `docs` project 中的修改：

```bash
cd docs
git status
git add repo/README.md repo/repo-command-guide.md
git commit -m "docs: add repo usage guide"
```

查看最近提交：

```bash
git log --oneline -5
```

返回 SDK 根目录后查看整体状态：

```bash
cd ..
repo status
```

## 8. 推送代码

### 8.1 使用普通 Git Push

如果远端仓库直接接受分支推送，可以进入对应 project 执行 `git push`。

首次推送当前分支：

```bash
cd docs
git push -u origin update_repo_docs
```

后续推送：

```bash
git push
```

推送到远端 `main` 分支：

```bash
git push origin HEAD:main
```

注意：直接推送 `main` 需要远端权限和团队流程允许。日常更建议推送 topic branch 后发起 PR/MR。

### 8.2 使用 Repo Upload

`repo upload` 适用于 Gerrit 类代码评审流程。如果远端没有配置 Gerrit review server，通常使用 `git push` 即可。

上传当前 topic branch 的改动：

```bash
repo upload docs
```

上传多个 project：

```bash
repo upload docs scripts
```

上传全部有 topic branch 的 project：

```bash
repo upload
```

如果执行 `repo upload` 报 review server 相关错误，说明当前仓库可能没有配置 Gerrit 流程，请改用普通 `git push`。

## 9. 拉取远端更新并处理冲突

推荐在 SDK 根目录同步：

```bash
repo sync
```

如果某个 project 有本地提交，`repo sync` 可能需要 rebase 或提示冲突。进入对应 project 处理：

```bash
cd linux
git status
```

常见处理方式：

```bash
git rebase origin/main
```

如有冲突：

```bash
git status
# 编辑冲突文件
git add <conflict-file>
git rebase --continue
```

放弃当前 rebase：

```bash
git rebase --abort
```

处理完成后回到 SDK 根目录：

```bash
cd ..
repo status
```

## 10. 批量执行命令

在所有 project 中执行同一条命令：

```bash
repo forall -c 'git status --short'
```

显示每个 project 的当前分支和最近提交：

```bash
repo forall -c 'echo "$REPO_PATH $(git branch --show-current) $(git log --oneline -1)"'
```

只在指定 project 执行：

```bash
repo forall linux u-boot -c 'git status --short'
```

常用环境变量：

| 变量 | 说明 |
| --- | --- |
| `REPO_PROJECT` | manifest 中的 project 名 |
| `REPO_PATH` | project 的本地路径 |
| `REPO_REMOTE` | project 使用的 remote 名 |
| `REPO_LREV` | manifest 解析后的本地 revision |
| `REPO_RREV` | manifest 中配置的 revision |

## 11. Manifest 操作

查看当前 manifest：

```bash
repo manifest
```

导出当前锁定版本：

```bash
repo manifest -r -o pinned-manifest.xml
```

用途：

- `repo manifest` 显示当前 workspace 使用的 manifest
- `repo manifest -r` 会把每个 project 固定到具体 commit，适合用于版本归档和可复现构建

如果 manifest 仓库有更新，同步 manifest 和所有 project：

```bash
repo sync
```

## 12. 清理和恢复

查看所有 project 的未提交修改：

```bash
repo status
```

丢弃某个 project 的未提交修改：

```bash
cd docs
git restore .
git clean -fd
```

注意：`git clean -fd` 会删除未跟踪文件，执行前先用 dry-run 查看：

```bash
git clean -fdn
```

如果某个 project 工作区损坏或同步异常，可尝试重新同步：

```bash
repo sync --force-sync docs
```

执行强制同步前，确认该 project 没有需要保留的本地提交或未提交修改。

## 13. 常用命令速查

| 场景 | 命令 |
| --- | --- |
| 初始化 workspace | `repo init -u git@github.com:huangliang367/manifest.git` |
| 同步全部代码 | `repo sync` |
| 并发同步 | `repo sync -j8` |
| 同步指定 project | `repo sync linux` |
| 查看整体状态 | `repo status` |
| 查看 project 列表 | `repo list` |
| 查看分支 | `repo branches` |
| 创建 topic branch | `repo start my_feature docs` |
| 批量执行命令 | `repo forall -c 'git status --short'` |
| 导出锁定 manifest | `repo manifest -r -o pinned-manifest.xml` |
| 提交代码 | `cd <project> && git add . && git commit -m "message"` |
| 推送分支 | `cd <project> && git push -u origin <branch>` |
| Gerrit 上传 | `repo upload <project>` |

## 14. 推荐日常流程

修改文档的典型流程：

```bash
repo sync docs
repo start update_docs docs
cd docs

# 修改文件
git status
git add repo/README.md repo/repo-command-guide.md
git commit -m "docs: update repo guide"
git push -u origin update_docs
```

修改内核的典型流程：

```bash
repo sync linux
repo start kernel_feature linux
cd linux

# 修改代码
git status
git add <files>
git commit -m "kernel: describe change"
git push -u origin kernel_feature
```

修改多个 project 的典型流程：

```bash
repo sync
repo start feature_x linux u-boot scripts

cd linux
git add <files>
git commit -m "kernel: part of feature x"

cd ../u-boot
git add <files>
git commit -m "u-boot: part of feature x"

cd ../scripts
git add <files>
git commit -m "scripts: support feature x"

cd ..
repo status
```

之后分别进入各 project 推送分支：

```bash
cd linux && git push -u origin feature_x
cd ../u-boot && git push -u origin feature_x
cd ../scripts && git push -u origin feature_x
```

