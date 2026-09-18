---
title: "SSH Permission denied 排查实录：VSCode 能连，Termux 密码却不行"
published: 2026-09-18
description: "同一台服务器，VSCode Remote-SSH 秒连，Termux 上用密码却 Permission denied？一次完整的排查思路复盘，顺便聊聊 ssh -v 的玄学。"
tags: ["SSH", "Linux", "Termux", "运维", "VSCode"]
category: Linux
draft: false
---

最近遇到一个很典型的问题：同一台内网服务器 `172.16.50.83`，在电脑上用 VSCode 的 Remote-SSH 连接一切正常，但在安卓 Termux 终端里用 `ssh root@172.16.50.83` 输密码，却总是 `Permission denied`。中间还出现了一个更玄学的现象——加上 `-v` 参数重试，居然就连上了。这篇文章把排查过程和结论整理成文。

## 先说结论

`Permission denied` 是**认证失败**，不是网络问题。VSCode 能连上，大概率根本不是靠密码，而是靠**密钥**；Termux 上用的密码方式恰好被服务器限制了。最经典的组合是这三条：

1. 服务器 `PasswordAuthentication no`，只接受密钥；
2. root 用户受 `PermitRootLogin prohibit-password` 限制——密钥能登 root，密码一律拒绝；
3. 压根没输对用户名。

## 为什么 VSCode 可以、密码不行

### VSCode 用的可能根本不是密码

VSCode Remote-SSH 读取的是 `~/.ssh/config`（Windows 上是 `C:\Users\<你>\.ssh\config`），里面通常长这样：

```
Host dev
    HostName 172.16.50.83
    User root
    IdentityFile ~/.ssh/id_rsa
```

只要配置了 `IdentityFile`，VSCode 走的就是**密钥认证**，跟你记不记得密码毫无关系。所以"VSCode 能连"并不能证明密码是对的，甚至不能证明服务器开了密码登录。

### 常见原因清单（按可能性排序）

| 原因 | 典型表现 |
|------|----------|
| `PasswordAuthentication no`，密码登录被禁用 | 报错为 `Permission denied (publickey)`，括号里根本没有 password |
| 用户名不一致 | Termux 直接 `ssh 172.16.50.83` 会用 Termux 本地用户名登录，而不是你 Windows 上的用户 |
| `PermitRootLogin prohibit-password` | root 允许密钥登录，密码怎么输都被拒——正是"VSCode 可以、密码不行"的经典场景 |
| 密码本身输错 | Termux 软键盘/输入法容易吃掉特殊字符、搞错大小写 |
| `AuthenticationMethods publickey,password` | 要求密钥 + 密码双因子，只给密码不够 |
| 账号被锁 / PAM 限制 | 密码正确依然拒绝，日志里有明确记录 |

### 怎么区分：看报错括号里的关键字

`Permission denied` 后面括号里列出的，是**服务器还愿意接受的认证方式**：

- `Permission denied (publickey)` —— 服务器不接受密码，别再试密码了，直接上密钥；
- `Permission denied (publickey,password)` —— 密码方式是开着的，问题出在用户名、密码本身或 root 限制。

## 两分钟定位法

### 第一步：Termux 上开详细日志

```bash
ssh -v root@172.16.50.83
```

重点看这两行输出：

```
debug1: Authentications that can continue: publickey
debug1: Next authentication method: password
```

第一行是服务器宣布"我还接受哪些方式"，第二行是客户端实际在用的方式。对照一下，问题基本就浮出水面了。

### 第二步：强制只用密码，排除密钥干扰

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password user@172.16.50.83
```

### 第三步：在电脑上看 VSCode 到底用什么连的

在 PowerShell 里执行：

```powershell
type C:\Users\<你>\.ssh\config
```

90% 的情况，答案就在这个文件里——`User` 是谁、`IdentityFile` 指向哪个私钥，一目了然。

### 第四步（有服务器权限的话）：直接看服务端

```bash
sudo sshd -T | egrep -i 'permitrootlogin|passwordauthentication|allowusers'
sudo tail -50 /var/log/auth.log    # CentOS 是 /var/log/secure
```

日志里会明确写着 `Failed password for invalid user ...` 还是 `User root not allowed because ...`，比客户端猜测准确得多。

## 解决方案

### 方案 A（推荐）：给 Termux 也配密钥

与其折腾密码，不如直接复用密钥方案：

```bash
pkg install openssh
ssh-keygen -t ed25519          # 一路回车
cat ~/.ssh/id_ed25519.pub
```

把输出的公钥追加到服务器的 `~/.ssh/authorized_keys`（可以从 VSCode 已连上的远程终端里直接粘贴进去）。再给 Termux 建一个和 PC 一样的 `~/.ssh/config`：

```
Host dev
    HostName 172.16.50.83
    User root
    IdentityFile ~/.ssh/id_ed25519
```

以后在手机上 `ssh dev` 就能直连，免密、安全、不用记密码。

### 方案 B：坚持用密码

修改服务器 `/etc/ssh/sshd_config`：

```
PasswordAuthentication yes
PermitRootLogin yes    # 如果用 root；更安全的做法是 prohibit-password + 密钥
```

然后重启服务：

```bash
systemctl restart sshd    # 或者 service ssh restart
```

### 方案 C：核对用户名

在 VSCode 的远程终端里敲 `whoami`，Termux 里就用 `ssh 那个用户名@172.16.50.83`，**别省略用户名**。

## 插曲：为什么"加上 -v 就连上了"

排查中途还出现过更诡异的情况：`ssh root@...` 失败，加上 `-v` 重试却成功了。先说答案——**`-v` 参数本身不改变任何协议行为**，它只是往终端打印调试日志，认证方式、密钥协商、用户名全都和它无关。真正的原因通常是：

1. **重试时条件已经变了**：前几次密码输错触发了 fail2ban 临时封禁，等敲带 `-v` 的命令时封禁刚好过期；或者服务器瞬时抖动、sshd 正在重启，第二次就好了。
2. **两次命令其实不一样**：第一次可能漏了用户名、密码输错。用 `history` 对比两条命令的实际内容，经常能直接发现差异。
3. **Termux 环境变化**：手机在两次尝试间切换了 Wi-Fi/热点/VPN——172.16.x.x 是内网地址，走移动数据必然连不上；或者后台挂着僵死的 ssh 会话（`pgrep ssh` 查一下）。

验证方法很简单，交替执行带 `-v` 和不带 `-v` 的相同命令各几次。如果两者都能稳定成功，说明 `-v` 无关，之前只是偶发。顺带一提：即使 `-v` 不影响结果，排查 SSH 问题时也建议**始终带上它**——失败原因基本都藏在 `Authentications that can continue:` 这几行日志里。

## 总结

- `Permission denied` = 认证失败，先看括号里允许的认证方式，再决定是配密钥还是查密码；
- "VSCode 能连" ≠ "密码正确"，VSCode 大概率在用 config 里的私钥；
- 内网 IP 连不上，先确认手机所在的网络能不能路由到；
- 服务端 `/var/log/auth.log` 永远是最终答案，有条件优先看它；
- 推荐的长期解法：**在 Termux 上生成密钥并部署到服务器**，密码登录能不开就不开。
