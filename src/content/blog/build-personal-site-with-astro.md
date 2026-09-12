---
title: '从零搭建免费个人博客 + 在线小工具网站（Astro + Cloudflare）'
description: '零成本建站全流程：Astro 官方模板起步，GitHub 管代码，Cloudflare 自动部署，纯前端小工具即写即上线。'
pubDate: '2026-09-13'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

一直想要一个自己的网站：能写博客，也能放一些顺手写的小工具。这篇文章记录从零到上线的完整过程，全部**免费**，不需要服务器，不需要备案。

## 技术选型

| 组件 | 选择 | 理由 |
|---|---|---|
| 框架 | [Astro](https://astro.build/) | 为内容站而生，构建快、SEO 好，又能在页面里写任意交互脚本（做工具） |
| 代码托管 | GitHub | 免费，且是部署触发器 |
| 部署 | Cloudflare（Git 集成） | 免费额度充足、不限带宽，海外免费托管里国内访问相对最好 |

> 国内访问速度参考：Cloudflare > GitHub Pages > Vercel（近年国内不稳定）。追求国内秒开再考虑国内云 + 备案。

## 第一步：用官方模板初始化

```sh
npm create astro@latest -- --template blog --no-install --no-git my-site
cd my-site && npm install && npm run dev
```

打开 `http://localhost:4321` 就能看到带文章列表、标签页、RSS、sitemap 的完整博客骨架。

> 💡 如果模板下载超时（从 GitHub 拉取），给命令加上代理环境变量再跑一次。

## 第二步：改成自己的站

几个关键文件：

- `src/consts.ts` —— 站点标题和描述
- `src/pages/index.astro` —— 首页（我改成了：介绍 + 最新文章 + 工具入口）
- `src/components/Header.astro` / `Footer.astro` —— 导航和页脚，把英文文案换成中文，导航加一个「工具」入口
- `astro.config.mjs` —— `site` 字段填最终线上域名（RSS 和 sitemap 依赖它）

**写博客**就是往 `src/content/blog/` 丢 Markdown 文件：

```markdown
---
title: '文章标题'
description: '一句话摘要'
pubDate: '2026-09-13'
heroImage: '../../assets/blog-placeholder-1.jpg'  # 可选
---

正文……
```

## 第三步：加在线小工具

Astro 做纯前端工具非常顺手——一个 `.astro` 页面 + 内联 `<script>`，构建后就是原生 JS，没有任何框架运行时负担。

固定模式（以后加新工具就两步）：

1. `src/pages/tools/` 下新建 `xxx.astro`，页面结构 + `<script>` 里写交互逻辑
2. 工具索引页和首页的 `tools` 数组里各加一张卡片

本站的 [JSON 格式化](/tools/json-formatter/)、[密码生成器](/tools/password-generator/)、[时间戳转换](/tools/timestamp/) 都是这个模式，全部在浏览器本地运行，无后端、零隐私顾虑。

## 第四步：推上 GitHub

```sh
git init -b main
git add -A && git commit -m "init"
gh repo create my-site --public --source=. --push   # 需要 gh CLI
```

没有 `gh` 也可以在网页上建好空仓库后 `git remote add origin` + `git push`。

## 第五步：Cloudflare 接管部署

1. 登录 [Cloudflare 控制台](https://dash.cloudflare.com) → **Workers 和 Pages** → **创建应用**
2. **导入 Git 仓库** → 授权 GitHub → 选中仓库
3. 构建配置：框架预设选 **Astro**（自动填 `npm run build` 和输出目录 `dist`；新版控制台会根据 `package-lock.json` 自动识别，没有预设下拉框也不影响）
4. 保存并部署，一分钟后拿到 `https://项目名.你的子域.workers.dev`

从此 `git push` = 自动构建部署。

## 建站之后

- **日常发布**：写 Markdown → `git push`，1-2 分钟后上线
- **想国内直连**：绑定自定义域名（Cloudflare 上直接加域，域名在注册商处把 DNS 指过去）
- **可扩展**：评论（giscus）、访问统计、更多工具，都是加文件的事

整套东西的持有成本几乎为零：没有服务器要维护，没有月账单，代码和内容都在自己手里。
