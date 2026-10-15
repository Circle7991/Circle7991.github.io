# Circle 的技术博客

使用 Hexo + Fluid 构建，通过 GitHub Actions 发布到 GitHub Pages。

网站地址：<https://circle7991.github.io/>

## 本地预览

安装 Node.js 22 LTS 与 Git，在仓库目录运行：

```bash
npm ci
npm run server
```

## 写文章

在 `source/_posts/` 中新增 Markdown 文件，顶部填写文章信息：

```yaml
---
title: 文章标题
date: 2026-10-15 18:00:00
categories:
  - Unity打包
tags:
  - Unity
---
```

也可以运行 `npm run new -- "文章标题"` 创建文章。

## 发布

提交并推送到 `main` 分支后，Actions 自动构建和发布。仓库 Settings → Pages 的 Source 使用 GitHub Actions。

## 配置

- `_config.yml`：网站标题、地址、文章链接。
- `_config.fluid.yml`：Fluid 主题与导航。
- `source/about/index.md`：关于页面。
- `.github/workflows/pages.yml`：自动发布。

主题：[Fluid](https://github.com/fluid-dev/hexo-theme-fluid)。博客文章的转载授权请联系作者。
