---
title: "统一管理 Markdown 内容并多平台发布的实践笔记"
date: 2026-02-21
aliases:
  - /zh/posts/统一管理 Markdown 内容并多平台发布的实践笔记/
tags: [内容创作, Git, Hexo, mdBook, Subtree, 博客部署]
categories: [技术实践]
---

# 统一管理 Markdown 内容并多平台发布的实践笔记

---

## 🧭 背景与目标

在日常创作中，我们常常会写很多 Markdown 格式的文章、笔记或教程。这些内容可能需要发布到多个平台，比如：

- 使用 **Hexo** 搭建的博客
- 使用 **mdBook** 构建的文档站
- 甚至是 **NotionNext** 这类基于 Notion 的博客系统

为了避免内容分散、重复维护，我希望实现以下三个目标：

1. **统一管理所有 Markdown 内容**：无论是博客、教程、文档，都集中在一个 Git 仓库中管理。
2. **支持多平台发布**：同一份内容可以根据需要发布到不同平台，格式和结构可适配。
3. **自动化同步与部署**：通过 Git 工具或 GitHub Actions 实现内容的同步与自动部署。

---

## 🧩 核心实现思路：版本管理与自动部署

为了实现内容的统一管理与多平台发布，整个系统的核心思路可以分为两个部分：

---

### 📦 1. 使用 `git subtree` 管理内容同步

我们使用 `git subtree` 来实现主仓库与各个发布仓库之间的子目录级同步。具体做法是：

- 在主仓库中维护所有内容（如 `shared/`、`hexo/`、`mdbook/` 等）
- 每个平台的发布仓库只同步主仓库中对应的子目录
- 使用 `git subtree push` 和 `git subtree pull` 实现双向同步

例如，将 `hexo/` 目录同步到 Hexo 博客仓库：

```bash
git subtree push --prefix=hexo https://github.com/yourname/hexo-blog.git main
```

这样可以保持内容的集中管理，同时又能灵活地将不同部分发布到不同平台。

---

### 🚀 2. 使用 GitHub Actions 实现自动部署

为了实现持续集成与部署（CI/CD），我们可以在发布仓库中配置 GitHub Actions。例如，对于 Hexo 博客，可以使用如下工作流,实现push自动编译和部署：

```yaml
name: Hexo Build and Deploy (GitHub Pages)

on:
  push:
    branches:
      - master
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm install

      - name: Generate static files
        run: npx hexo generate

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: './public'

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

只要你将内容同步到发布仓库并 push，GitHub Actions 就会自动构建并部署你的博客，无需手动操作。

---

通过 `git subtree` + GitHub Actions 的组合，我们可以实现：

- 内容集中管理
- 多平台分发
- 自动化部署

这正是现代内容创作者所需要的高效工作流！🍃

---

## 🧱 内容管理策略：文件夹优于分支

在管理多平台内容时，有两种常见方式：

- 使用 Git 分支（branch）分别管理不同平台的内容
- 使用文件夹（folder）在一个分支中组织不同平台的结构

经过实践，我选择了**文件夹管理**，原因如下：

| 对比维度 | 使用分支 | 使用文件夹 |
|----------|----------|------------|
| 内容复用 | ❌ 难以跨分支共享 | ✅ 可直接引用共享内容 |
| 操作复杂度 | ❌ 频繁切换分支，容易冲突 | ✅ 所有内容一目了然 |
| CI/CD 配置 | ❌ 每个分支需单独配置 | ✅ 可集中管理 |
| 本地预览 | ❌ 不便于同时预览多个平台 | ✅ 可并行运行多个平台服务 |

推荐的目录结构如下：

```
content/
├── shared/       # 原始 Markdown 内容
├── hexo/         # Hexo 博客格式
├── mdbook/       # mdBook 格式
```

---

## 🔀 内容同步策略：使用 git subtree

为了将某个子目录（如 `hexo/` 或 `mdbook/`）同步到另一个仓库进行部署，我使用了 `git subtree`，它支持将子目录与其他仓库的某个分支进行绑定和同步。


```bash
git subtree add --prefix=source/_posts https://github.com/yourname/markdown-src-repo.git hexo --squash
git subtree pull --prefix=source/_posts https://github.com/yourname/markdown-src-repo.git hexo --squash
```

---

## 🔧 内容迁移技巧

### ✅ 从 master 分支提取某个文件夹到其他分支：

```bash
git checkout hexo
git checkout master -- src/
git add src/
git commit -m "Import src/ from master"
```

### ✅ cherry-pick 某个提交（保留路径结构）：

```bash
git checkout hexo
git cherry-pick <commit-hash>
```

### ✅ 只提取文件内容，不保留路径结构：

```bash
git checkout hexo
git checkout master -- src/
mv src/* ./
rm -r src/
git add .
git commit -m "Flatten src/ into root"
```

或者：

```bash
git show master:src/a.md > a.md
git add a.md
git commit -m "Import a.md from master/src"
```

---

## 总结

通过统一的 Git 仓库 + 文件夹结构 + git subtree，同一个 Markdown 内容源可以灵活地分发到多个平台，既保证了内容一致性，又提升了维护效率。  
