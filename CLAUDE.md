# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Hexo 8 静态博客（中文内容，站点 <https://xuling.me>），主题为 Butterfly（git submodule），部署到自建服务器。

## 常用命令

hexo CLI 已全局安装，命令直接用 `hexo ...` 即可（package.json 里有等价的 pnpm scripts）。除依赖安装/更新外，以下命令均可直接使用：

```bash
hexo server                   # 本地预览 http://localhost:4000
hexo server --draft           # 预览时包含草稿
hexo clean                    # 清除缓存（db.json）和 public/
hexo generate                 # 生成静态站点到 public/
hexo deploy                   # 将 public/ 推送到部署仓库（需 SSH 访问服务器）

hexo new "文章标题"            # 新建文章 → source/_posts/（模板见 scaffolds/post.md）
hexo new draft "标题"          # 新建草稿 → source/_drafts/（默认不渲染）
hexo publish <文件名>          # 发布草稿（_drafts/ 移入 _posts/；参数是文件名 slug，不是标题）
```

完整发布流程：`hexo clean && hexo generate && hexo deploy`。

## 架构

- **`_config.yml`** — Hexo 站点配置：站点信息、permalink（`:year/:month/:day/:title/`）、部署目标。
- **主题 Butterfly** — `themes/butterfly` 是 git submodule（上游 jerryc127/hexo-theme-butterfly）。主题定制统一写在根目录 `_config.butterfly.yml`（Hexo 的 `_config.[theme].yml` 覆盖机制），不要直接修改 submodule 内的文件，否则 submodule 会变 dirty。
- **内容** — `source/_posts/*.md`，front-matter 包含 title / date / tags / categories。正文用中文，文件名用英文 kebab-case。
- **Butterfly 特殊页面** — `source/categories`、`source/tags`、`source/link` 各有 index.md（front-matter 设 `type`），删除会导致对应页面 404；友链数据在 `source/_data/link.yml`。
- **部署** — hexo-deployer-git 将生成的 public/ 推送到自建服务器 `git@47.96.85.100:/home/git/blog.git`（main 分支），服务器端负责托管；`.deploy_git/` 是它的工作目录。

## 注意事项

- `public/`、`db.json`、`.deploy_git/` 均为生成产物，已 gitignore，不要手动编辑或提交。
- `updated_option: mtime`：文章的 updated 时间取文件 mtime，修改旧文章会自动刷新其 updated。
- 不要自动安装或更新依赖：依赖由 Dependabot 每日自动提 PR 管理（见 .github/dependabot.yml），包管理器为 pnpm。
- pnpm 隔离 node_modules 下，主题（submodule）scripts 中 require 的包（如 hexo-util、moment-timezone）必须在根 package.json 显式声明为直接依赖，否则 hexo generate 报 `Cannot find module`。
