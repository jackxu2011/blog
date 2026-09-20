---
title: Claude Code 官方文档：用 Worktree 跑并行会话
date: 2026-09-01 13:48:56
tags:
  - claude
  - git
  - worktree
categories:
  - 编程
---

> 📖 **来源**：本文译自 Claude Code 官方文档 [Run parallel sessions with worktrees](https://code.claude.com/docs/en/worktrees)，作者 Anthropic。翻译基于 2026-09 版本，技术细节以英文原文为准，引用之处均已注明。
>
> ⏱️ 如果只是想用起来，看完下面的「一分钟上手」就够了；再往后的章节是恢复会话、自定义创建、subagent 隔离、排错时才需要查的细节。

## 🚀 一分钟上手

第一次用先花一分钟做两件事：忽略 worktree 目录、声明要带进 worktree 的本地文件。然后就是启动、退出。

### 一次性准备（仓库根目录）

```bash
# 1. 让主检出不把 worktree 内容显示为未跟踪文件
echo ".claude/worktrees/" >> .gitignore
```

```bash
# 2. 声明哪些被 gitignore 的本地文件要自动复制进每个新 worktree
#    只有「匹配模式 + 已被 gitignore」的文件才会被拷，用不到的行留着不生效
cat > .worktreeinclude <<'EOF'
# 通用：本地环境变量与凭据
.env
.env.local
.env.*.local
.secrets.*
config/secrets.json
secrets/*.json
service-account*.json
*.pem
*.key

# Java：依赖在 ~/.m2 与 ~/.gradle/caches，只需带本地配置
gradle.properties
local.properties
application-local.yml
application-secrets.yml
*.jks
*.p12

# Python：venv 用 uv sync 重建，只带本地覆盖配置
local_settings.py

# Go：模块与构建缓存在全局，只带本地运行配置
config.local.yaml
.air.toml
Makefile.local

# Node：node_modules 交给 pnpm install
.wrangler/
EOF
```

> ⚠️ 依赖目录和构建产物**一律不要**放进去（`.venv`、`node_modules`、`target/`、`build/`、`dist/`、`.gradle/`、`vendor/`）：它们内部写死了绝对路径，拷到新目录就是坏的，重建比复制更快。判断标准是三条同时满足：① 已被 gitignore、② 项目跑起来必需、③ 体积小且不含绝对路径。详见 [把 gitignored 文件带进 worktree](#把-gitignored-文件带进-worktree)。

> 💡 **`.claude/settings.local.json` 不用放进来**：worktree 里的会话会直接读写**主检出**根目录下的那一份（v2.1.211+），文件本身不需要出现在 worktree 里。放进 `.worktreeinclude` 只会多出一份不会被读取的副本。见 [worktree 与主检出共享什么](#worktree-与主检出共享什么)。

### 日常循环

```bash
# 终端 1：为这个任务建一个独立工作区并在其中启动 Claude
claude --worktree feature-auth      # 简写 claude -w feature-auth

# 终端 2：另一个任务，另起一个互不干扰的工作区
claude -w fix-login-bug

# 在新 worktree 里装一次依赖（它是全新检出）
# 干活 → 提交 → 退出时按提示保留或删除 worktree
# 回到主检出合并（自动创建的分支名为 worktree-<name>）
cd ~/my-project
git merge worktree-feature-auth
```

一个 worktree = 一个独立目录 + 一个独立分支 = 一个 Claude 会话。会话之间的文件修改互不可见，所以可以让一个会话写新功能、另一个修 Bug，同时推进。

## 核心概念

[git worktree](https://git-scm.com/docs/git-worktree) 是一个独立的工作目录，拥有自己的文件和分支，但与主检出共享同一份仓库历史和 remote。把每个 Claude Code 会话放进各自的 worktree，一个会话的编辑就永远不会碰到另一个会话的文件。

几点前提与定位：

- **必须有 git 仓库**。其他版本控制系统（SVN、Perforce、Mercurial）需要用 hook 替换掉内置的 git 逻辑，见后文。
- **桌面版自动隔离**：Desktop App 里每个新 session 都会自动获得自己的 worktree。
- **worktree 只是并行手段之一**：它解决的是*文件隔离*。[Subagents](https://code.claude.com/docs/en/sub-agents) 是在单个会话内部拆分工作，[跨会话消息](https://code.claude.com/docs/en/cross-session-messaging) 让不同 worktree 里的会话互相传递结论。三者的取舍见 [Run agents in parallel](https://code.claude.com/docs/en/agents)。

## 在 worktree 中启动 Claude

`--worktree`（或 `-w`）加一个名字，就会创建隔离工作区并在其中启动。默认行为：

| 项目 | 默认值 |
| --- | --- |
| 存放位置 | 仓库根目录下的 `.claude/worktrees/<name>/` |
| 分支名 | `worktree-<name>`（注意带 `worktree-` 前缀） |

在另一个终端换个名字再跑一次，就是第二个隔离会话。**省略名字**时 Claude 会自动生成一个可读的 slug，比如 `bright-running-fox`。

> ⚠️ **workspace trust**：交互式运行要求信任该目录。如果从没在这个目录跑过 Claude，先跑一次 `claude` 接受信任弹窗，否则 `--worktree` 会报错退出。`-p` 非交互模式跳过这个检查。
>
> 💡 官方建议把 `.claude/worktrees/` 加进 `.gitignore`，免得 worktree 内容在主检出里显示为未跟踪文件。

### 初始化 worktree 环境

worktree 是一份全新的检出，所以开发环境要在里面重新准备（让 Claude 装依赖，或自己在 `.claude/worktrees/` 下的目录里跑一遍）。想让 `.env` 这类被 gitignore 的文件自动带进每个新 worktree，用 [`.worktreeinclude`](#把-gitignored-文件带进-worktree)。

### 让 Claude 自己创建 worktree

会话过程中直接说「用一个 worktree 来做」，Claude 会调用 [`EnterWorktree`](https://code.claude.com/docs/en/tools-reference) 工具建一个。进入后还能直接切到 `.claude/worktrees/` 下的另一个 worktree（传目标路径给 `EnterWorktree`），原来那个留在磁盘上不动。

**安全边界**：当 Claude 要进入 `.claude/worktrees/` 之外的路径时，Claude Code 会先要求你批准——因为这次移动会把会话的工作目录、写权限和项目配置（`CLAUDE.md`、settings）一起带到那里。加 `EnterWorktree` 的 permission rule 或选「don't ask again」都不能跳过这个弹窗，只有 `bypassPermissions` 模式可以。

> ℹ️ **hook 里的路径不跟着 worktree 走**：进入 worktree 后，`${CLAUDE_PROJECT_DIR}` 仍然指向会话启动时的项目根目录，所以 `${CLAUDE_PROJECT_DIR}/.claude/hooks/xxx.sh` 这类命令还是在主检出里执行；真正跟着 Claude 移动的是 hook 输入 JSON 里的 `cwd` 字段（它等于 worktree 根目录，Claude 再 `cd` 时又跟着变）。需要 worktree 路径时读 `cwd`。

## 清理 worktree

退出一个交互式 worktree 会话时，Claude 会检查有没有「删掉就没了」的东西——改动过的文件、未跟踪文件、新提交——然后：

- **干净**：未命名的会话，直接自动删除 worktree 和分支；[命名过的](https://code.claude.com/docs/en/sessions#name-your-sessions)会话会先问你，方便你留着以后用
- **有东西**：弹出提示让你选保留还是删除。保留 = 目录和分支都留着，之后还能回来；删除 = 目录、分支连同里面的工作一起消失

**非交互（`-p`）运行不会清理**：没有退出弹窗，所以 worktree 和创建时加的锁都留在磁盘上，直到后续会话的定期清理（stale-lock sweep）释放。手动删除用 `git worktree remove`，如果 git 因为锁定拒绝，先 `git worktree unlock`。

**Windows**：删除 worktree 不会删掉它外部的文件。如果 worktree 里某个文件夹是指向别处的链接（NTFS junction 或目录符号链接），只删链接本身，保留目标文件夹。

## 恢复 worktree 会话

恢复一个曾在 worktree 里跑的会话时，Claude Code 会把会话送回那个 worktree。交互式恢复、`-p` 非交互下的 `--continue` / `--resume`、以及 Agent SDK 都是这个行为。回到 worktree 后，仍可以用 `ExitWorktree` 工具离开。

送回之前，Claude Code 会校验这个 worktree 依然是独立于主检出的工作区，校验不通过就不进。影响结果的因素：

- **启动目录**：从主检出或仓库的其他目录启动时，Claude Code 能重新进入它用 git 建在 `.claude/worktrees/` 下的 worktree（即使你就从里面启动）。但从*其他* worktree 里启动时，只有它能从该位置确认这个 worktree 的身份才会进入——worktree 本身是独立仓库、没有 git 元数据、或者你从 `git worktree add` 建的子目录里启动，都会失败，所以这些情况请从主检出启动
- **`--fork-session`**：分叉出的会话从你的启动目录开始，原会话的 worktree 不受影响
- **worktree 已被删除**：在当前启动目录继续，告诉你 worktree 不在了，并清除该会话的 worktree 绑定

进出 Claude Code 用 git 创建的 worktree 时，transcript 会跟着走——会话记录被挪到新工作目录下（和 `/cd` 一样），`/desktop` 和 `--resume` 也能在那里找到它；退出时同样移回。hook 创建的 worktree，transcript 仍留在启动目录。

## 隔离是怎么被强制执行的

会话被隔离在 worktree 里时，Claude Code 会拦截下面这几类落到主检出的工具调用。无论你是用 `--worktree` 启动、Claude 用 `EnterWorktree` 进入、还是恢复的 worktree 会话，规则一致；该会话派生的**所有 subagent 同样适用**，包括后台运行的会话。

四条检查：

1. **文件编辑**：`Edit` / `Write` / `NotebookEdit` 目标落在主检出 → 拦截
2. **命令工作目录**：Bash / PowerShell / Monitor 命令的 cwd 解析到主检出（或无法验证其在主检出之外）→ 拦截
3. **git 重定向**：命令通过 `git -C`、`--git-dir`、`GIT_DIR` / `GIT_WORK_TREE` 环境变量，或先 `cd` 进主检出再跑 git，把 git 操作引回主检出 → 拦截
4. **命令形态**：无法静态验证「一定留在 worktree 内」的 shell 结构（如 brace expansion、delimiter 未加引号的 heredoc）→ 拦截，即使命令根本不碰 git。Claude Code 会告诉 Claude 怎么改写（比如拆成多个独立命令），**这条检查无法关闭**

检查范围针对你启动 Claude Code 的那个仓库，也覆盖「被链接的 worktree 所链接的主检出」。PowerShell 命令只应用第 2 条。每次拦截，Claude 看到的是一条指名 worktree、并说明如何继续的 tool error。

## 用 worktree 隔离 subagent

让 subagent 各跑自己的 worktree，避免并行编辑冲突：直接对 Claude 说「给你的 agents 用 worktree」，或者给自定义 subagent 在 `.claude/agents/` 的 frontmatter 里写死 `isolation: worktree`：

```markdown
---
name: refactorer
description: Applies mechanical refactors across many files
isolation: worktree
---

Apply the requested refactor across every affected file, then run the tests
and report the results.
```

每个 subagent 拿到一个临时 worktree，跑完没有改动就被自动删除；有改动的会留在磁盘上，等到下面的定期清理在「不会丢工作」时移除。Subagent 的 worktree 和 `--worktree` 使用同一个[基线分支](#选基线分支)，即除非把 `worktree.baseRef` 设成 `"head"`，否则从仓库默认分支切出。

### 清理 subagent 与后台会话的 worktree

Claude Code 会周期性 sweep，删除超过 [`cleanupPeriodDays`](https://code.claude.com/docs/en/settings-reference#cleanupperioddays) 的 subagent / [后台会话](https://code.claude.com/docs/en/agent-view#how-file-edits-are-isolated) worktree。以下情况会被**保留**：

- 里面还有活：改动过的文件、未跟踪文件、未推送的提交
- 无法判定仓库配置定义了哪些 filter driver（对应后文三种阻止创建的情况）
- 属于一个你没转入后台的 `--worktree` 会话（无论多久）
- 是你自己用 `git worktree add` 建的——即使你在里面跑了 `--worktree <name>` 会话并把它转后台

原理是 Claude Code 会往每个它创建的 worktree 的 git 元数据里写一个 marker，没有 marker 的一律不清理。Agent 运行期间 Claude Code 持有 `git worktree lock`，防止并发清理把它删掉；进程已退出的会话留下的锁会被 sweep 释放（被 kill 的后台会话不会永久锁死），但你手动 `git worktree lock` 的锁它从不动。

要清理被 sweep 保留下来的 worktree，跑 `git worktree remove`（有未提交改动或未跟踪文件时加 `--force`），被锁拒绝时先 `git worktree unlock`。

## 自定义 worktree 创建

默认配置（建在 `.claude/worktrees/`、从默认分支切出、只检出被跟踪的文件）覆盖大多数场景，下面这些用来改默认值。

### 选基线分支

在 settings 里设 [`worktree.baseRef`](https://code.claude.com/docs/en/settings-reference#worktree)，只接受两个值：

| 值 | 行为 |
| --- | --- |
| `"fresh"`（默认） | 从远端默认分支（通常是 `main`）切出，起点与远端一致 |
| `"head"` | 从当前本地 `HEAD` 切出，带上未推送的提交和 feature 分支状态；要在进行中的工作上跑 subagent 时用这个。注意在某个 worktree 内部，`"head"` 指的是**该 worktree 的** `HEAD` |

不能把它设成某个分支名。要从特定已存在分支开始，直接用 git 手动[创建 worktree](#手动管理-worktree)。

`"fresh"` 模式下 Claude Code 会尽量让 `origin/HEAD` 保持新鲜：仓库超过 24 小时没 fetch 过就补一次（最多等 5 秒），失败则用本地缓存的 ref；没有配置 remote 或拿不到 `origin/HEAD` 时，退回到当前本地 `HEAD`。

```json
{
  "worktree": {
    "baseRef": "head"
  }
}
```

### 从 PR / MR 切出

给 `--worktree` 传 `#` 加编号、GitHub PR URL 或 GitLab MR URL（如 `https://gitlab.com/group/repo/-/merge_requests/123`），Claude Code 会从 `origin` 拉取该变更的 head commit，并在 `.claude/worktrees/pr-<number>` 建 worktree。**记得加引号**，否则 shell 会把 `#` 当注释起点：

```bash
claude --worktree "#1234"
```

URL 里只取编号，且始终从仓库的 `origin` 拉取，按 host 决定 fetch 路径：github.com 用 `pull/<number>/head`；gitlab.com 用 `merge-requests/<number>/head`；GitHub Enterprise、私有化 GitLab 及其他 host 先试前者再试后者。

### 把 gitignored 文件带进 worktree

worktree 是新检出，主仓库里的 `.env`、`.env.local` 之类不会在里面。在**项目根目录**放一个 `.worktreeinclude` 文件即可让 Claude Code 建 worktree 时自动复制：

```text
.env
.env.local
config/secrets.json
```

语法同 `.gitignore`，且**只有既匹配模式、又被 gitignore 的文件**会被复制，被跟踪的文件不会被重复一份。`**/` 开头的模式配合整体被忽略的目录时有额外细节（要看目录名是否出现在路径里），复杂场景建议直接查原文。

这适用于所有 Claude Code 用 git 创建的 worktree：`--worktree`、subagent、桌面版并行会话。用 `WorktreeCreate` hook 时，改为在 hook 脚本里自己复制。

#### 各语言该往 `.worktreeinclude` 里放什么（译注补充）

> 以下是译者的实践建议，官方文档只给了 `.env` / `.env.local` / `config/secrets.json` 这个例子，并没有按语言分类。

先明确三条筛选标准，全部满足才放进来：

1. **已经被 gitignore**：官方规则，被跟踪的文件永远不会被复制，写了也没用
2. **项目跑起来必需**：拷一份过去就能用，不拷就得每次手配
3. **体积小、不含绝对路径**：这条把依赖目录和构建产物挡在外面——它们拷过去要么是坏的，要么白白拖慢创建速度

反面清单（**不要**放进去）：`.venv`、`node_modules`、`target/`、`build/`、`dist/`、`vendor/`、`.gradle/`。Python 虚拟环境里 `pyvenv.cfg`、activate 脚本、entry-point 的 shebang 全都写死了绝对路径，复制到新目录后解释器和控制台脚本直接失效；`node_modules` 则是体积巨大且包管理器自己会做硬链接/符号链接优化，重建比复制更快。Go 的模块缓存和构建缓存本来就在 `$HOME` 下，天然跨 worktree 共享，更不需要复制。

##### 通用（不分语言）

```text
.env
.env.local
*.pem
.secrets.*
```

##### Java（Maven / Gradle）

依赖在 `~/.m2/repository` 或 Gradle 的 `~/.gradle/caches` 全局仓库里，新 worktree 直接复用，`mvn` / `./gradlew` 首次构建就能命中，所以只放配置：

```text
gradle.properties
local.properties
application-local.yml
application-secrets.yml
*.jks
*.p12
```

`gradle.properties` 常因含私服账号而被 gitignore，不拷过去就会出现「本地能构建、worktree 里拉不到依赖」的诡异现象；`local.properties`（Android SDK 路径）和 `application-local.yml`（本地数据库连接）同理。

##### Python（uv / poetry / venv）

虚拟环境一律重建（`uv sync` 有全局缓存，通常几秒），只放环境变量与密钥：

```text
.env
.env.local
local_settings.py
secrets/*.json
service-account*.json
```

`local_settings.py` 是 Django 项目常见的本地覆盖文件；`service-account*.json` 是 GCP/AWS 凭据。注意 poetry 若把 venv 建在项目内 `.venv`，同样不要复制。

##### Go

Go 基本不需要这个文件——依赖和构建缓存都在全局。真正常用的是本地运行配置：

```text
.env
config.local.yaml
.air.toml
Makefile.local
```

`config.local.yaml` 指本地开发用的应用配置（数据库地址等），`air.toml` 是 air 热重载的本地配置。`vendor/` 若存在通常是被提交的（已跟踪），用不上。

##### Node / npm / pnpm

```text
.env
.env.local
.env.development.local
.npmrc
.wrangler/
```

`.npmrc` 排第一：私服 registry token 通常都 gitignore，不拷过来 `install` 会 401。Vite / Next 读 `.env*.local`；`node_modules` 交给 `pnpm install`（内容寻址 + 硬链接，秒级完成）。

回到本博客这个仓库，`node_modules/` 和 `.deploy_git/` 都不该进 `.worktreeinclude`（前者是依赖目录、后者是生成产物，`hexo` 全局命令在哪个 worktree 里都能跑），真要用 worktree 并行改文章，只需要放 `.env`（如果你有）。

### 重名即复用

传一个目录已存在的名字给 `--worktree`，会**打开已有的那个 worktree** 而不是新建。若基线是默认的 `"fresh"`，且同时满足下列条件，重开的 worktree 会重置到仓库默认分支，而不是停在旧的 tip：

- 没有未提交改动、没有未跟踪文件
- 还在 Claude Code 为它创建的那个分支上
- 自己没有额外提交，或者它的 PR/MR 已合并且远端分支已删除（Claude Code 能从 git 状态识别：远端分支已不存在 + 所有 commit 都已在默认分支上）

其他情况一律在旧 tip 重开，比如：不满足上面任一条件、状态验证不了、`worktree.baseRef` 是 `"head"`、名字是 PR/MR 引用。

### 用 hook 完全接管创建逻辑

配置 [`WorktreeCreate`](https://code.claude.com/docs/en/hooks#worktreecreate) hook 可以替换掉默认的 `git worktree` 逻辑，包括把 worktree 放到 `.claude/worktrees/` 以外的位置。

## worktree 与主检出共享什么

除了各自的文件和分支，worktree 还会共享仓库的 `.git` 目录、项目级插件、以及已保存的权限授权：

- **`.git` 目录**：worktree 里的 git 命令写入主仓库共享的 `.git`，sandboxing 也放行这些写入，所以在 worktree 里开沙箱照样能 `git commit`
- **插件**：在主检出以 project scope 安装的插件，同仓库的 worktree 里也会加载，不必逐个 worktree 重装
- **权限授权**：worktree 会话读写的是**主检出**根目录下的 `.claude/settings.local.json`（v2.1.211+），这个文件不需要存在于 worktree 里，所以也不用进 `.worktreeinclude`。在里面选「Yes, and don't ask again」的 Bash 规则会写进该文件，因此对本仓库的所有 worktree 生效，worktree 删了也还在（Windows 及其他不使用仓库根目录的场景，规则跟着那个 worktree 走）

以上三条与创建方式无关：`--worktree`、`git worktree add`、桌面版，都适用。

## 手动管理 worktree

需要检出特定已存在分支、或把 worktree 放到仓库外时，直接用 git：

```bash
# 新分支建 worktree
git worktree add ../project-feature-a -b feature-a

# 基于已有分支建 worktree（fix-issue-456 换成仓库里已存在的分支）
git worktree add ../project-bugfix fix-issue-456

# 在里面启动 Claude
cd ../project-feature-a
claude

git worktree list                     # 查看
git worktree remove ../project-feature-a   # 用完删除
```

完整参数见 [Git worktree 文档](https://git-scm.com/docs/git-worktree)。

## 非 git 版本控制

需要给 SVN、Perforce、Mercurial 等做隔离时，用 [`WorktreeCreate` / `WorktreeRemove`](https://code.claude.com/docs/en/hooks#worktreecreate) hook 提供自定义的创建与清理逻辑。注意 hook 替换掉了默认 git 行为，所以用 hook 时 `.worktreeinclude` **不生效**，本地配置文件要在 hook 脚本里自己复制。

下面这个 `WorktreeCreate` hook 从 stdin 的 JSON 里用 `jq` 读名字，checkout 一份 SVN 工作副本，并把目录路径打印出来供 Claude Code 作为工作目录（配置写进 `settings.json`）：

```json
{
  "hooks": {
    "WorktreeCreate": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'NAME=$(jq -r .name); DIR=\"$HOME/.claude/worktrees/$NAME\"; svn checkout https://svn.example.com/repo/trunk \"$DIR\" >&2 && echo \"$DIR\"'"
          }
        ]
      }
    ]
  }
}
```

再配一个 `WorktreeRemove` hook 负责会话结束时清理。

## 排错要点

创建、启动时进入、恢复时回到 worktree 这三个环节，Claude Code 会报下面这些错：

- **启动时进不去 worktree**：报错并指出路径，退出码 1。常见原因是 `WorktreeCreate` hook 打印的不是它真正创建的目录，或者目录建好后被删了
- **路径是符号链接**：`.claude`、`.claude/worktrees` 或 worktree 目录本身是 symlink 时，拒绝创建并报出该路径。删掉软链再试
- **LFS 文件只有指针**：`git lfs install --local` 把 filter 写进了仓库自己的 `.git/config`，而 Claude Code 创建 worktree 时会**跳过仓库自定义的 filter driver**（因为 filter 本质是 shell 命令，任何能写仓库的东西都可能塞一个进去）。解法：在 worktree 里跑 `git lfs pull`。同理适用于其他写在仓库 config 里的 filter driver。另外三种「连 worktree 都不建」的情况：`.git/config` 读不了（权限）、某个 filter driver 名字里含 `=` 或换行、`.git/config` 里用了 `includeIf` 条件引入（把引入的设置直接写进该文件再试；全局 config 里的 `includeIf` 不触发）
- **`Refusing to use <path> as an isolation worktree`**：Claude Code 检查了该目录的 git 身份后拒绝采用。多数是它的 `.git` 指向主检出，或通过 `core.worktree` 让 git 把工作树解析回主检出——在这种目录里跑 `git reset --hard` 会打到主检出上。按报错尾句对号入座：让你 `launch from the parent checkout` 就从主检出重新启动；`it contains the protected checkout` 说明被拒目录是你主检出的父目录（比如 home 目录），**别删**，改 worktree 路径；涉及主检出自身 git 元数据读不了的，修主检出而**不是**重建 worktree；`its recorded path has a network spelling` 则必须换到本地路径重建
- **恢复时没回到 worktree**：会给出 `Your worktree <path> no longer exists`（目录没了，无隔离继续，绑定已清除，无需处理）、`Could not verify ... this time`（多为瞬时问题，绑定保留，再 resume 一次）、`Did not re-enter ...`（判定不安全，绑定清除）、`Could not re-enter ...`（多半是你从 worktree 内部启动，绑定保留）四类提示之一，具体恢复方式对齐上面那条 refusal 的说明

## 延伸阅读

- [Subagents](https://code.claude.com/docs/en/sub-agents)：把委派出去的工作放进这些隔离检出
- [Cross-session messaging](https://code.claude.com/docs/en/cross-session-messaging)：让不同 worktree 里的会话互通结论
- [Agent teams](https://code.claude.com/docs/en/agent-teams)：自动协调多个 Claude 会话
- [Manage sessions](https://code.claude.com/docs/en/sessions)：命名、恢复、切换会话
- [Desktop 并行会话](https://code.claude.com/docs/en/desktop#work-in-parallel-with-sessions)：桌面版基于 worktree 的多会话

---

> 原文：[Run parallel sessions with worktrees](https://code.claude.com/docs/en/worktrees) · [完整文档索引](https://code.claude.com/docs/llms.txt)
> 官方文档更新频繁，文中提到的默认路径、清理行为、版本相关细节请以英文原文为准。
