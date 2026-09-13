---
title: '路线 C：网页终端——浏览器直连家里的 Claude Code'
description: 'ttyd + tmux + 内网穿透，把家里电脑的 AI Agent 变成一个网址：手机/公司浏览器打开就能用，断线任务不死，还带"只能看不能动"的公司安全口。'
pubDate: '2026-09-13'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

[上一篇](/blog/remote-control-home-agent/)写了路线 A（ToDesk 远程桌面）和路线 B（Tailscale + SSH）。用了一天后我给自己加了个需求：**能不能连 App 都不装，手机浏览器打开一个网址就用？** 顺手还解决了两个老问题——SSH 断线任务陪葬、公司电脑想看家里进度又怕数据回流。

这就是路线 C：**ttyd 网页终端 + tmux 会话层 + 内网穿透**。

## 架构

```
手机/公司浏览器
   │ HTTPS（带密码认证）
   ▼
内网穿透入口（二选一：Tailscale Funnel / Cloudflare 隧道）
   │
   ▼
ttyd（电脑上的网页终端服务，只监听本机）
   │
   ▼
tmux 会话 "web"（断线不死 + 多端同屏 + 只读镜像）
   │
   ▼
bash → claude
```

## 第一步：ttyd，把终端变成网页

[ttyd](https://github.com/tsl0922/ttyd) 是单文件的网页终端服务，Windows 版开箱即用：

```sh
# 官方 Releases 下载 ttyd.win32.exe 后：
ttyd.exe -c admin:你的强密码 -W -m 2 -i 127.0.0.1 -p 7681 "C:\Program Files\Git\bin\bash.exe" --login
```

参数拆解，每个都有安全含义：

- `-c admin:密码`：HTTP Basic 认证，没有它终端就是裸奔
- `-W`：允许输入（**不加就是只读**——这个特性后面有大用）
- `-m 2`：最多 2 个并发连接（小写 m！大写 `-M` 不存在，还会把参数解析错位，端口悄悄回退默认值，极难排查）
- `-i 127.0.0.1`：只监听本机回环，公网入口是唯一通道

### Windows 大坑：服务方式运行时 claude 找不到登录态

想开机自启就要用计划任务以 SYSTEM 账户跑。结果 `claude` 一启动就弹登录界面——因为 **Node 读的是 `USERPROFILE`，而 SYSTEM 账户的 USERPROFILE 指向系统目录**，claude 读不到你用户目录下的 `~/.claude` 配置。解决：启动脚本里显式设两个变量再拉起 ttyd：

```bat
set HOME=C:\Users\你的用户名
set USERPROFILE=C:\Users\你的用户名
ttyd.exe ...
```

## 第二步：tmux，会话不死 + 只读镜像

裸 ttyd 有个致命伤：**浏览器一断（手机锁屏、标签页关闭），它就把终端里的进程杀了**——跑一半的 Claude 任务直接陪葬。解法是让终端跑在 tmux 里，ttyd 只负责"接入"：

- 读写口：`tmux new -A -s web`（有会话就接入，没有就新建）
- 只读口：`tmux attach -t web -r`（`-r` = 只读接入，**看的是同一个画面**，但敲键盘无效）

效果三连：

1. **断线不死**：网页关了任务继续跑，重连回到原画面，滚屏历史都在
2. **只读镜像**：公司浏览器打开只读口 = 实时看家里 Claude 干活的直播屏，物理上无法输入 → 公司数据不可能流向家里电脑
3. **多端同屏**：SSH 里 `tmux -S ~/.tmux-web.sock new -A -t web` 加入同一会话，ToDesk、手机、浏览器看的是同一块屏

### Git Bash 装 tmux 的三个坑

1. **版本必须匹配运行时**：Git Bash 本质是精简版 MSYS2，新 tmux 配旧 Git 的 `msys-2.0.dll` 会静默退出（exit 127 无任何报错）。要么升级 Git 到最新版再装新 tmux，要么找同年代的 tmux 包
2. **terminfo 缺失**：tmux 默认 `default-terminal` 是 `tmux-256color`，Git Bash 的 terminfo 库里没有，报 `missing or unsuitable terminal`。配置里改成存在的条目：`set -g default-terminal "screen-256color"`
3. **工作目录**：SYSTEM 下 tmux 会话默认出生在 `C:\Windows\System32`，claude 在系统目录启动会弹信任确认。接入脚本第一行先 `cd "$HOME"`

## 第三步：公网入口，二选一（或都要）

### 方案 1：Tailscale Funnel（固定域名，需代理）

已经在用 Tailscale 的话这是零新增方案，把 tailnet 内的服务安全暴露到公网：

```sh
tailscale funnel --bg 7681    # 得到 https://你的机器名.你的tailnet.ts.net/
```

自签 HTTPS、配置随 tailscaled 持久化、重启不变。缺点：`ts.net` 域名国内直连不通，手机要先开代理。

### 方案 2：Cloudflare 快速隧道（国内直连，域名随机）

```sh
cloudflared.exe tunnel --url http://127.0.0.1:7681
# 输出里会打印 https://随机词-随机词-随机词.trycloudflare.com
```

**国内实测可直连**（trycloudflare 走 Cloudflare 边缘，和被 DNS 污染的 workers.dev 命运不同），不需要账号。缺点：域名每次重启都变，把输出重定向到日志文件备查：

```sh
grep -o "https://.*" ~/bin/cloudflared/quicktunnel.log | tail -1
```

我的选择：**两个都开**——日常手机走快速隧道（直连零依赖），公司走 Funnel 固定域名（加书签）。

## 第四步：开机自启

一条命令注册计划任务（以 SYSTEM 身份、开机启动）：

```powershell
schtasks /create /tn ttydRW /tr "C:\path\to\ttyd-run.bat" /sc onstart /ru SYSTEM /rl HIGHEST /f
```

ttyd 读写、ttyd 只读、cloudflared 三个任务同理。配套一个看门狗（每 5 分钟检查 Tailscale 在线状态，卡死自动重启服务），细节见[改造实录](/blog/home-server-devlog/)。

## 安全要点

- **密码必须专用且够长**：网页终端的密码等于电脑的钥匙，别复用任何旧密码。注意 Funnel/隧道流量经过 Tailscale/Cloudflare 的服务器中转，理论上中转方可见——这是用随机强密码而非短密码的原因
- **只读口给不可信环境**：公司电脑、公共设备一律只读口，读写口只给自己
- **止损要快**：怀疑泄露就改 ttyd 的 `-c` 密码重启服务（30 秒），旧连接全部失效；不放心再 `tailscale funnel off` 直接关公网入口
- **私钥和密码永远不进文章、不进网盘、不进群聊**

## 三条路线最终对比

| | A：ToDesk | B：Tailscale+SSH | C：网页终端 |
|---|---|---|---|
| 手机要装 | ToDesk App | Tailscale + ConnectBot | **什么都不装** |
| 网络 | 国内直连 | 点对点直连 | 直连或代理（双入口） |
| 断线续跑 | ❌ | `claude -c` 续对话 | **tmux 原画面续跑** |
| 公司安全 | ❌ 不建议 | 密钥即身份 | **只读口零回流** |
| 图形界面 | ✅ | ❌ | ❌ |

我的最终形态：**C 做日常（手机浏览器随开随用）、B 做备用（SSH 稳定可靠）、A 做图形兜底（要看浏览器/软件时）**。一套系统，三种姿势。
