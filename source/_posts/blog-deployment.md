---
title: Hexo 博客 GitHub Actions 部署指南
date: 2026-11-10
tags: [指南]
categories: [技术]
description: 本文主要讲解如何通过 GitHub Actions 实现 Hexo 博客的部署流程，以及常见的排错和错误原因
---

## 标准生成步骤

### 配置 `package.json`
确保 `scripts` 中显式使用 `npx`，并在 `devDependencies` 中精确锁定 `hexo-cli` 版本。

```json
{
  "name": "your-blog-name",
  "version": "1.0.0",
  "private": true,
  "hexo": { "version": "" },
  "scripts": {
    "build": "npx hexo generate",
    "clean": "npx hexo clean"
  },
  "dependencies": {
    "hexo": "^7.3.0",
    "hexo-theme-butterfly": "^5.7.0",
    "...": "其他插件"
  },
  "devDependencies": {
    "hexo-cli": "4.3.0" 
  }
}
```
> **注意**：`hexo-cli` 建议使用确切版本（如 `4.3.0`），避免使用 `^` 或 `~` 导致自动升级引发兼容性问题。

### 配置 `deploy.yml`
完整的 GitHub Actions 工作流配置（保存至 `.github/workflows/deploy.yml`）：

```yaml
name: Deploy Blog

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4

      - name: 安装 Node.js (LTS)
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: 清理并重装依赖
        run: |
          rm -rf node_modules package-lock.json
          npm install
          npm install hexo-cli@4.3.0 --save-dev --save-exact

      - name: 编译生成静态网页
        run: |
          npx hexo clean
          npx hexo generate
          if [ ! -d "public" ]; then
            echo "::error::public 目录未生成！构建失败。"
            exit 1
          fi

      - name: 配置 Pages
        uses: actions/configure-pages@v5

      - name: 上传编译产物
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: 发布到 GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 配置 GitHub 仓库
进入仓库 **Settings > Pages**，将 **Build and deployment** 下的 **Source** 设置为 **GitHub Actions**。

---

## 更改主题配置

### 准备工作

* 首先在 `node_modules` 内找到 `*-theme-*` 目录内的 `_config.yml` 备份
* 此时进入到原本的配置文件内其中找到需要的字段进行更改

### 更改博客头像

* 在 `_config.yml` 内找到 `avatar` 字段将准备好的头像图片地址添加到此处
* 例如 `icon.png` 在 `source/imgs/` 目录下 此时字段应该为：`img: /imgs/icon.png`

### 自定义样式

* 在 `_config.yml` 内找到 `inject` 字段将准备好的css文件地址添加到此处
* `inject` 用于将css或javascript文件添加到头部或底部
* 例如取消图像的hover旋转

```yml
inject:
  head:
    - <link rel="stylesheet" href="./css/disable.css">
  bottom:
    # - <script src="xxxx"></script>
```

```css
.avatar-img img:hover {
    -webkit-transform: rotate(0);
    -moz-transform: rotate(0);
    -o-transform: rotate(0);
    -ms-transform: rotate(0);
    transform: rotate(0);
}
```

## 常见报错与排错指南

### ❌ 报错 1：`The butterfly theme could not be found`
*   **原因**：Actions 环境中未正确安装主题依赖。
*   **解决**：确保 `package.json` 的 `dependencies` 中包含 `hexo-theme-butterfly`，且本地执行 `npm install` 后将 `package-lock.json` 提交至仓库。

### ❌ 报错 2：`tar: public: Cannot open: No such file or directory`
*   **原因**：Hexo 编译未生成 `public` 目录。
*   **解决**：
    1. 检查 `deploy.yml` 是否添加了 `if [ ! -d "public" ]; then exit 1; fi` 校验，防止未生成目录时进入下一步。
    2. 展开“编译生成静态网页”日志，查看 `npx hexo generate` 是否报错。若无输出，参考报错 3。

### ❌ 报错 3：`Usage: hexo <command>` (无实际生成日志)
*   **原因**：`hexo-cli` 版本不兼容、上下文丢失或未调用本地依赖。
*   **解决**：
    1. 将 `hexo-cli` 降级并锁定至 `4.3.0`。
    2. `scripts` 中使用 `npx hexo ...`。
    3. 在 CI 中先执行 `rm -rf node_modules package-lock.json` 并重新 `npm install`，排除缓存干扰。

### ❌ 报错 4：`Dependencies lock file is not found`
*   **原因**：`actions/setup-node` 开启了 `cache: 'npm'`，但仓库根目录缺少 `package-lock.json`。
*   **解决**：在本地执行 `npm install` 生成 `package-lock.json` 并提交至仓库；或临时移除 `setup-node` 中的 `cache: 'npm'` 配置。

---

## 通用检查清单
1. **目录结构**：`_config.yml`、`package.json`、`source` 必须在仓库**根目录**。
2. **忽略文件**：`.gitignore` 中必须包含 `public/`、`node_modules/`、`db.json`，但**不能**忽略 `source/`。
3. **文章时间**：若文章 `date` 晚于服务器当前时间，Hexo 会拒绝渲染。可在 `_config.yml` 中设置 `future: true` 解决。
