# 我的小站

个人博客 + 在线小工具网站。基于 [Astro](https://astro.build/) 构建，托管在 Cloudflare（Git 集成自动部署）。

- 线上地址：https://personal-website.1215833383.workers.dev
- 代码仓库：https://github.com/songcong123/personal-website

## 如何写一篇博客

在 `src/content/blog/` 下新建 Markdown 文件（文件名即 URL），开头写 frontmatter：

```markdown
---
title: '文章标题'
description: '一句话摘要'
pubDate: '2026-09-13'
heroImage: '../../assets/blog-placeholder-1.jpg'   # 可选
---

正文支持所有常用 Markdown 语法……
```

## 如何新增一个小工具

1. 在 `src/pages/tools/` 下新建 `xxx.astro` 页面（参考 `json-formatter.astro`：页面结构 + 内联 `<script>` 交互逻辑，纯浏览器运行）
2. 在 `src/pages/tools/index.astro` 和 `src/pages/index.astro` 的 `tools` 数组里各加一张卡片

## 日常命令

| 命令           | 作用                          |
| :------------- | :---------------------------- |
| `npm run dev`  | 本地开发（localhost:4321）    |
| `npm run build`| 构建产物到 `./dist/`          |
| `npm run preview` | 本地预览构建结果           |

## 发布流程

```sh
git add -A
git commit -m "新文章：xxx"
git push
```

推送后 Cloudflare 会自动构建并部署（约 1~2 分钟）。

> 注意：本仓库已配置走本地代理推送（`git config http.proxy`），因为直连 GitHub 不通。
> `*.workers.dev` 域名在国内直连受限，访问线上站点需代理，后续可绑定自定义域名改善。

## 目录结构

```text
├── public/               # 静态资源（favicon 等）
├── src/
│   ├── assets/           # 图片、字体
│   ├── components/       # Header、Footer 等组件
│   ├── content/blog/     # 博客文章（Markdown）
│   ├── layouts/          # 文章页布局
│   ├── pages/
│   │   ├── index.astro   # 首页
│   │   ├── blog/         # 博客列表 / 文章页 / 标签页
│   │   ├── tools/        # 在线小工具
│   │   └── rss.xml.js    # RSS 订阅
│   ├── styles/           # 全局样式
│   └── consts.ts         # 站点标题、描述
└── astro.config.mjs      # 站点配置（域名、集成）
```

## Credit

基于 [Astro 官方 blog 模板](https://github.com/withastro/astro/tree/main/examples/blog)（theme 设计来自 [Bear Blog](https://github.com/HermanMartinus/bearblog/)）修改。
