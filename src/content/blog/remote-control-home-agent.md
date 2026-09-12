---
title: '远程操作家里电脑上的 AI Agent：两条路线完整教程'
description: '路线 A：ToDesk 远程桌面开箱即用；路线 B：Tailscale + SSH + 密钥登录，手机随时随地连回电脑终端驱动 Claude Code。'
pubDate: '2026-09-13'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

在电脑上跑 AI 编程 Agent（如 Claude Code）很爽，但人总不在电脑前。这篇文章记录两条远程操作家里电脑 Agent 的路线，从简单到专业，都是免费方案，含安卓端安装配置全流程。

## 路线 A：远程桌面（ToDesk，10 分钟搞定）

想要「和坐在电脑前一模一样」的体验，直接用国产远程桌面：

1. 电脑和手机都装 [ToDesk](https://www.todesk.com/)（个人免费，国内直连）
2. 电脑端注册登录，**设置 → 安全设置 → 安全密码** 设一个强密码（这是无人值守远程的固定密码）
3. 设置里勾选**开机自启**，电脑重启后依然可连
4. 手机 App 登录同一账号 → 设备列表点电脑 → 输安全密码 → 连接

优点：零门槛，图形界面完整可用。缺点：手机小屏操作桌面终端体验一般——这就要路线 B 出场了。

## 路线 B：Tailscale + SSH（为 CLI Agent 而生）

架构：**手机 → Tailscale 虚拟局域网 → 家里电脑的 SSH 服务 → Git Bash → `claude`**。

断线不丢任务：Agent 跑在电脑上，SSH 断了重连后 `claude -c` 继续上一次对话。

### 1. 电脑端：装 SSH 服务 + Tailscale

用管理员 PowerShell 一次装完（Windows 10/11 通用）：

```powershell
# 安装并启动 SSH 服务
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Set-Service sshd -StartupType Automatic
Start-Service sshd

# 防火墙放行 22 端口（安装后一般已有，没有就补一条）
if (-not (Get-NetFirewallRule -Name "OpenSSH-Server-In-TCP" -ErrorAction SilentlyContinue)) {
  New-NetFirewallRule -Name "OpenSSH-Server-In-TCP" -DisplayName "OpenSSH Server (sshd)" `
    -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
}

# SSH 登录后默认 shell 改成 Git Bash（claude 在 PATH 里，连上即用）
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell `
  -Value "C:\Program Files\Git\bin\bash.exe" -PropertyType String -Force
```

Tailscale 去 [官网](https://tailscale.com/download) 装 Windows 版，装好后：

```sh
tailscale login   # 弹出链接，浏览器里用 GitHub 账号授权
tailscale ip -4   # 拿到这台机器的虚拟内网 IP，形如 100.x.x.x
```

### 2. 电脑端：配置手机专用的 SSH 密钥

```sh
# 生成一把手机专用密钥（独立持有，泄露可单独吊销）
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_phone -N "" -C "my-phone"
```

⚠️ **关键坑**：Windows 管理员账户的公钥不读 `~/.ssh/authorized_keys`，要写进系统级文件并设置严格 ACL（管理员 PowerShell）：

```powershell
$pub = Get-Content "$env:USERPROFILE\.ssh\id_ed25519_phone.pub" -Raw
Set-Content -Path "C:\ProgramData\ssh\administrators_authorized_keys" -Value $pub.TrimEnd() -Encoding ascii
icacls "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r /grant "Administrators:F" /grant "SYSTEM:F"
Restart-Service sshd
```

本机自测：

```sh
ssh -i ~/.ssh/id_ed25519_phone localhost 'claude --version'
```

### 3. 手机端：Tailscale + SSH 客户端

**Tailscale**：应用商店搜，搜不到就在电脑上（挂代理）从 [GitHub Releases](https://github.com/tailscale/tailscale-android/releases) 下 APK 传到手机安装。登录**同一账号**，连上后顶部出现 VPN 图标。

**SSH 客户端（ConnectBot）**：开源免费，官方发布页下载 APK（需代理）：[GitHub Releases](https://github.com/connectbot/connectbot/releases)，选最新版 `ConnectBot-vx.x.x-google.apk`。部分国内应用商店（应用宝等）也收录了 ConnectBot，可以先搜搜看。

手机配置四步：

1. 把电脑上的私钥文件 `id_ed25519_phone`（注意是**没有 `.pub` 后缀**的那个）内容发到手机（微信文件传输助手即可）
2. ConnectBot → 菜单 ⋮ → **管理密钥** → **导入** → 粘贴私钥全文
3. 主机列表 `+` 新建：地址填 `你的电脑用户名@100.x.x.x`，端口 22，认证选刚导入的密钥
4. 点击连接，看到 Git Bash 提示符即成功

### 4. 远程驱动 Agent

```bash
claude           # 新对话
claude -c        # 继续上一次对话（断线重连就靠它）
claude --resume  # 列出历史会话挑选恢复
```

下班路上手机发个任务，地铁里看它跑，回家电脑上 `claude -c` 无缝续上。

## 安全要点

- **私钥永远不进公开位置**：不传网盘公开链接、不上网站、不发群里。它等于你家电脑的钥匙
- **安全密码要强**（ToDesk 那条密码同理）
- **SSH 只在 Tailscale 网内可达**：不暴露公网端口，扫描器根本看不见你的 22 端口
- 手机丢失：删掉 Tailscale 授权 + 换 SSH 密钥对即可止损，不用动电脑

## 两条路线怎么选

| | 路线 A（ToDesk） | 路线 B（Tailscale+SSH） |
|---|---|---|
| 上手时间 | 10 分钟 | 40 分钟 |
| 适合 | 偶尔远程、要图形界面 | 高频使用终端 Agent |
| 手机体验 | 小屏操桌面 | 原生终端，极顺手 |
| 长任务 | 保持远程会话 | `claude -c` 断点续跑 |

我的用法：日常 B 路线，需要看浏览器/图形软件时 A 路线，两者互补。
