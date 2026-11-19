# git

本仓库对应 GitHub 上的 [`Mnn666-666/git`](https://github.com/Mnn666-666/git)。

## 开始使用

把项目文件放进这个目录，然后：

```bash
git add .
git commit -m "描述你的改动"
git push
```

---

# 本机 Git 连接配置说明

## 为什么这么配

原来的配置是 HTTPS + 本地代理，`http.proxy` 指向 `http://127.0.0.1:7892`。代理关掉后端口成了死端口，于是报：

```
Failed to connect to github.com:443 over proxy 127.0.0.1 after 0 ms
```

删掉代理后发现问题不只在代理：直连 `github.com:443` 时通时不通，DNS 解析出来的 IP 会被轮换封锁，勉强通了还会被 TCP RST 中断。

最终方案：**SSH over 443**。GitHub 官方支持把 SSH 跑在 443 端口上，用于绕过 22 端口被防火墙挡住的情况。这条链路不依赖具体 IP、全程加密、不受 HTTPS 干扰。

```
git push  →  ssh  →  ssh.github.com:443  →  GitHub
```

不需要开任何代理或加速器。

## 当前配置一览

| 项目 | 值 |
|---|---|
| remote | `git@github.com:Mnn666-666/git.git` |
| 传输 | SSH，走 `ssh.github.com:443` |
| 密钥类型 | ed25519 |
| 密钥路径 | `C:\Users\yeee\.ssh\id_ed25519` |
| 私钥密码 | 有（`aes256-ctr` 加密） |
| ssh-agent | 服务 `Running / Automatic`，启动脚本自动加载 |

相关文件：

```
~/.ssh/config        把 github.com 指向 ssh.github.com:443
~/.ssh/known_hosts   预置 GitHub 官方主机指纹，防中间人
~/.ssh/id_ed25519    私钥（有密码）
~/.ssh/id_ed25519.pub 公钥（已添加到 GitHub）
~/.ssh/autoload.log  启动脚本的日志
```

`~/.ssh/config` 内容：

```
Host github.com
  HostName ssh.github.com
  Port 443
  User git
```

## 换机器时的一次性配置

### 1. 生成密钥

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.ssh" | Out-Null
ssh-keygen -t ed25519 -C "你的邮箱" -f "$env:USERPROFILE\.ssh\id_ed25519"
```

> **踩过的坑**：不要在 PowerShell 5.1 里写 `-N ''` 想设空密码。
> 5.1 会把空字符串参数丢掉，导致后面的 `-q` 被当成密码值，密码变成 `-q`。
> 让 ssh-keygen 交互式提示，直接回车表示不设密码。

### 2. 写 ssh config

```powershell
@"
Host github.com
  HostName ssh.github.com
  Port 443
  User git
"@ | Set-Content "$env:USERPROFILE\.ssh\config" -Encoding ascii
```

### 3. 预置官方主机指纹

GitHub 官方公布的指纹见
<https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints>。

```powershell
$key = 'AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl'
@(
  "github.com ssh-ed25519 $key",
  "ssh.github.com ssh-ed25519 $key",
  "[ssh.github.com]:443 ssh-ed25519 $key"
) | Set-Content "$env:USERPROFILE\.ssh\known_hosts" -Encoding ascii
```

> 为什么要写三条：OpenSSH 实际按 `HostName:Port`（即 `[ssh.github.com]:443`）查
> known_hosts，只写 `github.com` 会报 `Host key verification failed`。

### 4. 添加公钥到 GitHub

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub" | Set-Clipboard
```

到 <https://github.com/settings/keys> → New SSH key → 类型选 **Authentication Key** → 粘贴 → 添加。

> 必须贴**公钥**（单行，以 `ssh-ed25519` 开头）。贴成私钥会报
> `Key is invalid. You must supply a key in OpenSSH public key format`。

### 5. 启用 ssh-agent 并加载密钥

```powershell
# 需要管理员权限
Set-Service ssh-agent -StartupType Automatic
Start-Service ssh-agent

# 普通权限
ssh-add "$env:USERPROFILE\.ssh\id_ed25519"
```

### 6. 验证

```powershell
ssh -T git@github.com
```

期望输出 `Hi 你的用户名! You've successfully authenticated...`。

### 7. 切换 remote

```powershell
git remote set-url origin git@github.com:Mnn666-666/git.git
```

如果 `ssh -T` 成功但 `git push` 仍报 publickey 错误，说明 git 调用的 ssh 与命令行不是同一个：

```powershell
git config --local core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"
```

## 日常使用

Windows 的 ssh-agent **不会把密钥持久化到磁盘**，重启后缓存清空。

开机时会运行启动脚本 `启动文件夹\ssh-agent-load.cmd`，弹窗提示输入一次密码，之后本次会话内所有 push 免密。

手动加载（任何时候都可以用）：

```powershell
ssh-add "$env:USERPROFILE\.ssh\id_ed25519"
ssh-add -l    # 查看已加载的密钥
ssh-add -D    # 清空
```

## 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| `Permission denied (publickey)` | agent 里没有密钥，或 GitHub 上没有对应公钥 | `ssh-add` 一次；核对指纹 |
| `Host key verification failed` | known_hosts 里缺少 `[ssh.github.com]:443` 条目 | 见「3. 预置官方主机指纹」 |
| `Key is invalid...` | 贴成了私钥，或复制时带进了换行 | 重新复制公钥那一行 |
| `Key is already in use` | 同一把公钥已存在于某个 GitHub 账号 | 检查是否重复添加 |
| 网页打不开 GitHub，但 git 正常 | hosts 里固定的 IP 失效了 | 改 IP 或删掉 hosts 那行，**不影响 git** |

验证连通性：

```powershell
Test-NetConnection ssh.github.com -Port 443 -InformationLevel Quiet
```

## 安全注意

- 私钥有密码，文件权限仅当前用户可读（`icacls` 核实过：SYSTEM / Administrators / 当前用户）。
- `known_hosts` 预置了官方指纹，首次连接不会盲信。
- 密码建议记在密码管理器里。**SSH 密钥的密码无法找回**，忘记只能重新生成一对新密钥、在 GitHub 上删旧加新（代码不受影响）。
- 不要用第三方 Git 加速镜像，凭据会经过他人服务器；也不要关闭 `http.sslVerify`。

## 排障记录（2026-11-18 ~ 11-19）

| 现象 | 根因 | 处理 |
|---|---|---|
| `over proxy 127.0.0.1 after 0 ms` | `http.proxy` 指向已关闭的 `7892` 端口 | 删除 `http.proxy` |
| `after 21105 ms: Could not connect` | DNS 返回的 GitHub IP 被封锁（SYN 超时约 21 秒） | 改 hosts 固定 IP |
| `Recv failure: Connection was reset` | HTTPS 流量被 RST 中断 | 改走 SSH over 443 |
| `Host key verification failed` | known_hosts 只写了 `github.com`，实际按 `[ssh.github.com]:443` 查 | 补全三个条目 |
| `Enter passphrase` 但没设过密码 | PowerShell 5.1 吞掉 `-N ''`，`-q` 被当成密码 | `ssh-keygen -p` 重置密码 |
| `Key is invalid` | 粘贴了私钥 | 贴公钥单行 |
