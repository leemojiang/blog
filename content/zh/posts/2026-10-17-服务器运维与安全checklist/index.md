---
title: "服务器运维与安全检查手册"
date: 2026-10-17
draft: false
math: false
layout: single
ShowToc: true
tags:
  - Linux
  - 服务器
  - 运维
  - 服务器安全
  - Docker
categories:
  - 服务器运维
description: >-
  面向基础薄弱读者的 Linux 服务器运维手册，整理安全检查、登录排查、端口防护、日志监控和 Docker 部署命令。
---
这份笔记主要记录我平时维护 Linux 服务器时常用的命令。目标是：先确认服务器现在是什么状态，再做配置或清理；每次改完后，检查服务是否正常。

> 命令需要根据服务器发行版稍作调整。Ubuntu/Debian 常用 `apt`，Rocky Linux/AlmaLinux/Fedora 常用 `dnf`。下面的 `<用户名>`、`<服务器IP>`、`<端口>` 需要换成自己的实际值。带 `sudo` 的命令通常需要管理员权限。

## 0. 操作前先确认

- [ ] 确认自己连的是哪台机器、当前用户是谁、系统是什么版本。

```bash
whoami
hostname
cat /etc/os-release
```

- [ ] 远程修改 SSH、防火墙或网络配置时，先保留当前 SSH 窗口，再开一个新窗口测试登录。确认新窗口可以登录后，再关闭旧窗口。
- [ ] 修改配置前先备份文件；修改后先检查语法，再重启服务。
- [ ] 可能删除数据或中断服务的命令，先看清楚目标、影响和备份。

备份一个配置文件的例子：

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

`cp` 是复制；第一个参数是原文件，第二个参数是备份文件。

## 1. 防御的前提是识别攻击

### 1.1 查看 CPU 和进程

```bash
top
```

- `top`：动态查看 CPU、内存和进程占用。按 `q` 退出。
- `htop`：更容易浏览的交互式进程查看器；没有安装时先安装。

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install -y htop

# Rocky Linux / AlmaLinux / Fedora
sudo dnf install -y htop
```

```bash
ps aux
ps aux | grep '[n]ginx'
```

- `ps`：查看进程；`a` 显示其他用户的进程，`u` 用易读格式显示用户和资源占用，`x` 包含没有终端的后台进程。
- `|` 把左边命令的结果交给右边命令。
- `grep` 按文字筛选；`[n]ginx` 可以避免把 `grep nginx` 自己也显示出来。

### 1.2 查看登录记录和失败登录

```bash
last -n 20
lastb -n 20
lastlog
```

- `last`：查看成功登录记录；`-n 20` 表示只显示最近 20 条。
- `lastb`：查看失败登录尝试。它统计的是记录条数，不等于独立攻击者人数；日志是否存在、是否有权限读取，也取决于系统配置。
- `lastlog`：查看各用户最近一次登录时间。部分新系统没有这个命令，或未启用对应记录；可以用 `last` 和系统日志补充查看。

查看 SSH 登录失败日志：

```bash
# Ubuntu / Debian 常见路径
sudo grep -i "Failed password" /var/log/auth.log

# Rocky Linux / AlmaLinux / Fedora 常见路径
sudo grep -i "Failed password" /var/log/secure

# 使用 systemd 日志的系统，也可以查看 ssh 服务日志
sudo journalctl -u ssh -u sshd --since "24 hours ago"
```

- `grep` 是文本搜索命令；`-i` 表示忽略大小写。
- `journalctl` 查看 systemd 日志；`-u` 指定服务，`--since` 指定起始时间。
- 日志里出现失败登录不一定代表服务器已被攻破；要结合来源 IP、时间、是否有成功登录和后续进程一起判断。

### 1.3 查看端口和网络连接

优先用 `ss`；`netstat` 是旧工具，在新系统中可能没有安装。

```bash
sudo ss -lntup
sudo ss -tnp
sudo lsof -i :22
```

- `ss` 查看 socket（网络连接和监听端口）。`-l` 只看监听中的端口，`-n` 用数字显示地址和端口，`-t` 看 TCP，`-u` 看 UDP，`-p` 显示关联进程。
- `sudo ss -tnp` 查看 TCP 连接；`-t` 为 TCP，`-n` 显示数字，`-p` 显示进程。
- `lsof -i :22` 查看正在使用 22 端口的程序；把 `22` 换成想检查的端口。
- `netstat -lntup` 在已安装 `net-tools` 的系统上也能查看监听端口；含义与上面的选项大致相同。

查找某个端口是否被占用：

```bash
sudo ss -lntup | grep ':3306'
```

### 1.4 查看磁盘、内存和系统负载

```bash
df -h
free -h
uptime
```

- `df -h`：查看各文件系统空间；`-h` 用 GB、MB 等易读单位显示。
- `free -h`：查看内存和 swap 使用情况；`-h` 同样表示易读单位。
- `uptime`：查看开机时间、运行时长和系统负载。负载需要结合 CPU 核心数和历史情况判断，不是单独的故障结论。

找出当前目录下占空间较大的项目：

```bash
sudo du -sh /var/log /var/lib/docker /opt 2>/dev/null
```

- `du` 查看目录占用；`-s` 只显示汇总，`-h` 使用易读单位。
- `2>/dev/null` 把权限不足等错误信息丢弃，不影响正常结果。

### 1.5 发现异常后怎么排查

- [ ] 记录异常出现的时间和现象，例如 CPU 飙高、磁盘写满、陌生端口监听。
- [ ] 用 `top` 或 `ps aux` 找到高占用进程。
- [ ] 用 `ss -lntup` 和 `lsof -i :<端口>` 确认程序监听了什么端口。
- [ ] 用 `journalctl` 或系统认证日志查看对应时间段的记录。
- [ ] 确认进程、服务和文件属于哪个软件，再决定是否停止或删除；不要看到陌生名字就直接删除。

查看最近一小时系统错误日志：

```bash
sudo journalctl -p err --since "1 hour ago"
```

`-p err` 只看错误级别；`--since` 指定起始时间。

## 2. SSH 登录安全

### 2.1 本地生成密钥

私钥保存在自己的电脑上，不能发给别人；服务器保存公钥。下面是在 Windows PowerShell 或 macOS/Linux 本地终端运行的例子：

```bash
ssh-keygen -t ed25519 -C "your_email_or_name"
```

- `ssh-keygen`：生成 SSH 密钥对。
- `-t ed25519`：选择 Ed25519 密钥算法。
- `-C`：给密钥加一个备注，方便以后辨认。
- 默认文件通常是 `~/.ssh/id_ed25519`（私钥）和 `~/.ssh/id_ed25519.pub`（公钥）。

指定密钥文件名的例子（Windows）：

```powershell
ssh-keygen -t ed25519 -f C:\Users\Administrator\.ssh\my_server_key -C "server1"
```

- `-f` 指定密钥保存路径；生成的公钥文件会在相同路径后加 `.pub`。

### 2.2 把公钥安装到服务器

如果本地有 `ssh-copy-id`，可以用它安装公钥：

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 22 <用户名>@<服务器IP>
```

- `-i` 指定要安装的公钥文件。
- `-p` 指定 SSH 端口；默认端口是 22。
- `<用户名>@<服务器IP>` 表示用哪个服务器账号连接。

Windows 没有 `ssh-copy-id` 时，可以先查看并复制公钥内容：

```powershell
Get-Content $env:USERPROFILE\.ssh\my_server_key.pub
```

登录服务器后，把复制的整行公钥追加到目标用户的 `~/.ssh/authorized_keys`。注意这里是追加，不能覆盖已有内容：

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "把本地 .pub 文件中的整行内容粘贴到这里" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

- `mkdir -p` 创建目录；目录已经存在时不报错。
- `chmod 700` 设置目录权限：只有该用户可以读、写、进入。
- `>>` 把内容追加到文件末尾；单个 `>` 会覆盖原文件。
- `chmod 600` 设置授权文件权限：只有该用户可以读、写。

本地也可以用 `scp` 传公钥文件，但要先传到临时文件，再登录服务器追加，避免覆盖已有授权列表：

```bash
scp -P 9753 ~/.ssh/id_ed25519.pub <用户名>@<服务器IP>:/tmp/new_key.pub
```

- `scp` 通过 SSH 复制文件；`-P`（大写）指定 SSH 端口。
- 上传到 `/tmp/new_key.pub` 后，在服务器登录对应用户并执行：

```bash
cat /tmp/new_key.pub >> ~/.ssh/authorized_keys
rm /tmp/new_key.pub
```

### 2.3 测试密钥登录

在本地新开一个终端窗口测试：

```bash
ssh -i ~/.ssh/my_server_key -p 9753 <用户名>@<服务器IP>
```

- `-i` 指定私钥文件。
- `-p` 指定服务器 SSH 端口。
- 确认能登录后，再考虑关闭密码登录。不要先关掉当前唯一可用的会话。

### 2.4 禁止密码登录和 root 直接登录

先确保普通用户可以通过密钥登录，并且该用户有可用的管理方式。然后在服务器上备份并编辑 SSH 配置：

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
```

确认或设置以下项：

```text
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
```

- `PubkeyAuthentication yes` 允许公钥登录。
- `PasswordAuthentication no` 禁止 SSH 密码认证。
- `PermitRootLogin no` 禁止 root 直接通过 SSH 登录。`yes` 会允许 root 登录，不符合这里的安全目标。
- 配置文件里如果同一选项出现多次，实际生效值可能受配置顺序和 `Include` 文件影响；改完要检查最终配置。

> !`PermitRootLogin no`需要慎重, 因为如果没有其他管理员用户，可能会把自己锁在服务器外面。先确认普通用户可以登录，并且有 sudo 权限。

检查配置并重启 SSH：

```bash
sudo sshd -t
sudo systemctl restart sshd
sudo systemctl status sshd --no-pager
```

- `sshd -t` 检查 SSH 服务端配置语法；无输出通常表示语法通过。
- `systemctl restart` 重启服务。
- `status` 查看服务状态；`--no-pager` 不进入分页界面。
- Ubuntu/Debian 上服务名有时是 `ssh`，如果 `sshd` 找不到就改用 `ssh`。

可查看 SSH 服务最终采用的部分配置：

```bash
sudo sshd -T | grep -E '^(pubkeyauthentication|passwordauthentication|permitrootlogin) '
```

### 2.5 修改 SSH 端口（可选）

改端口只能减少自动扫描带来的噪声，不能替代密钥登录和防火墙。先在防火墙和云厂商安全组放行新端口，再改 SSH 配置；保留旧连接并测试新端口成功后，再关闭旧端口。

```text
Port 9753
```

`Port` 后面是 SSH 服务监听的端口号。改完按上一节执行 `sshd -t`，再重启 SSH 服务。

本地测试：

```bash
ssh -p 9753 <用户名>@<服务器IP>
```

### 2.6 查看登录用户和 SSH 服务

```bash
who
whoami
sudo systemctl status sshd --no-pager
```

- `who` 查看当前登录会话。
- `whoami` 查看当前 shell 的用户名。
- `systemctl status` 查看服务状态；Ubuntu/Debian 可能需要把 `sshd` 改为 `ssh`。

## 3. 用户和权限管理

### 3.1 不用 root 直接运行日常程序

查看本机用户列表：

```bash
cat /etc/passwd
```

`/etc/passwd` 记录系统账号信息，不会显示用户密码。普通用户与系统服务账号都会出现在列表里。

创建普通用户：

```bash
sudo adduser eel
```

`adduser` 创建用户；`eel` 是新用户名。不同发行版也可能使用 `useradd`，它通常需要另外设置家目录和密码。

切换到该用户并加载其登录环境：

```bash
su - eel
```

- `su` 切换用户。
- `-` 表示加载目标用户的登录环境，包括家目录和环境变量。

给用户 sudo 权限（Ubuntu/Debian）：

```bash
sudo usermod -aG sudo eel
```

- `usermod` 修改用户属性。
- `-aG sudo` 把用户追加到 `sudo` 用户组；`-a` 表示追加，`-G` 指定附加组。不要漏掉 `-a`，否则可能替换掉用户原有的附加组。
- Rocky Linux/AlmaLinux 等系统通常使用 `wheel` 组：`sudo usermod -aG wheel eel`。
- 重新登录后组权限才会完整生效。

进入 root 管理 shell：

```bash
sudo -i
```

`sudo` 以管理员权限执行命令；`-i` 模拟 root 的登录 shell。只在需要时使用，完成后输入 `exit` 退出。

### 3.2 给每位用户单独配置 SSH 公钥

每个用户都应使用自己的密钥对。服务器把该用户的公钥放在该用户家目录下的 `~/.ssh/authorized_keys`；不要共用别人的私钥。

管理员为新用户准备 SSH 目录的例子：

```bash
sudo install -d -m 700 -o newuser -g newuser /home/newuser/.ssh
sudo install -m 600 -o newuser -g newuser /dev/null /home/newuser/.ssh/authorized_keys
```

- `install -d` 创建目录。
- `-m 700` 或 `-m 600` 指定权限。
- `-o` 指定所有者，`-g` 指定所属组。
- 把公钥追加到 `/home/newuser/.ssh/authorized_keys` 后，确认目录和文件的所有者仍是 `newuser`。

### 3.3 查看用户组和权限

```bash
id eel
groups eel
ls -ld /home/eel /home/eel/.ssh
```

- `id` 查看用户 ID 和所属组。
- `groups` 查看组成员关系。
- `ls -ld` 查看目录本身的权限、所有者和所属组；`-l` 长格式，`-d` 显示目录本身而不是目录内容。

## 4. 防火墙、端口与服务暴露

### 4.1 先确认服务器实际监听的端口

```bash
sudo ss -lntup
```

只开放确实需要从公网访问的服务端口。例如 Web 服务常用 80/443，SSH 常用 22 或自定义端口。数据库端口通常只给本机或私有网络访问。

### 4.2 Ubuntu / Debian 使用 UFW 的常见操作

```bash
sudo ufw status verbose
sudo ufw allow 9753/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

- `ufw` 是简单防火墙管理工具。
- `status verbose` 查看规则和状态。
- `allow 9753/tcp` 允许 TCP 9753 端口；端口要换成实际 SSH 端口。
- 在启用防火墙前，先允许当前 SSH 端口，避免把自己锁在服务器外面。

需要启用时：

```bash
sudo ufw enable
```

删除规则的例子：

```bash
sudo ufw delete allow 3306/tcp
```

### 4.3 Rocky Linux / AlmaLinux 等使用 firewalld 的常见操作

```bash
sudo firewall-cmd --state
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-port=9753/tcp
sudo firewall-cmd --reload
```

- `--state` 查看防火墙是否运行。
- `--list-all` 查看当前区域规则。
- `--permanent` 把规则持久保存。
- `--add-port=9753/tcp` 添加 TCP 端口规则。
- `--reload` 重新加载持久规则。

云服务器通常还有云厂商安全组。主机防火墙和安全组都要检查；其中任何一层没有放行，外部都无法访问。

### 4.4 Docker 容器不要随意暴露公网端口

Compose 示例：

```yaml
services:
  mysql:
    image: mysql:8.4
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE:-auth}
      MYSQL_USER: ${MYSQL_USER:-auth}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:?MYSQL_PASSWORD is required}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?MYSQL_ROOT_PASSWORD is required}
    ports:
      - "127.0.0.1:${MYSQL_HOST_PORT:-3306}:3306"
```

- `image` 指定要运行的镜像和版本。
- `restart: unless-stopped` 表示容器退出后自动重启，除非管理员主动停止。
- `environment` 设置容器环境变量。
- `${MYSQL_DATABASE:-auth}` 表示变量未设置时使用 `auth` 默认值。
- `${MYSQL_PASSWORD:?...}` 表示变量未设置时让 Compose 报错，并提示密码是必需的。
- `127.0.0.1:主机端口:容器端口` 只把端口绑定到服务器本机回环地址，公网无法直接连这个映射端口。
- 若写成 `3306:3306`，Docker 通常会绑定到所有网络接口，可能让数据库暴露到公网。

Docker 发布端口时可能绕过 UFW 的部分规则；不能只看 UFW 状态就认定容器端口安全。还要检查 Compose 的端口绑定、云安全组和 `ss -lntup`。Docker 官方也特别提醒了防火墙规则的影响：[Docker 与防火墙](https://docs.docker.com/engine/install/debian/)。

## 5. 更新、日志与基础防护

### 5.1 更新系统软件

Ubuntu / Debian：

```bash
sudo apt update
sudo apt upgrade
```

- `apt update` 更新软件包索引。
- `apt upgrade` 安装可升级的软件包；执行前阅读将要安装和移除的内容。

Rocky Linux / AlmaLinux / Fedora：

```bash
sudo dnf check-update
sudo dnf upgrade
```

- `dnf check-update` 检查可更新软件；它有时会用非零退出码表示发现更新，不一定是命令失败。
- `dnf upgrade` 安装更新。

### 5.2 查服务状态和日志

```bash
sudo systemctl status docker --no-pager
sudo journalctl -u docker --since "1 hour ago"
sudo journalctl -u docker -f
```

- `systemctl status` 查看服务状态。
- `journalctl -u docker` 查看 Docker 服务日志。
- `--since "1 hour ago"` 只看最近一小时。
- `-f` 持续跟随新日志，按 `Ctrl+C` 退出。

系统日志常用查看方式：

```bash
sudo journalctl -p warning --since today
sudo journalctl -b -1
```

- `-p warning` 查看 warning 及更严重级别的记录。
- `-b -1` 查看上一次开机期间的日志；日志是否保留取决于系统配置。

### 5.3 可选：使用 Fail2ban 降低重复撞密码

Fail2ban 会读取日志，并按规则临时封禁反复失败的来源 IP。它是辅助措施，不能代替 SSH 密钥、及时更新和防火墙。

```bash
# Ubuntu / Debian
sudo apt install -y fail2ban

# Rocky Linux / AlmaLinux：先确认启用了合适的软件仓库
sudo dnf install -y fail2ban

sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```

- `enable` 设置开机启动。
- `--now` 同时立即启动服务。
- `fail2ban-client status` 查看保护状态；具体 jail 需要根据系统日志和配置确认。

## 6. GitHub 与应用部署

### 6.1 安装 Git

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install -y git

# Rocky Linux / AlmaLinux / Fedora
sudo dnf install -y git

git --version
```

- `&&` 表示前一个命令成功后才运行后一个命令。
- `-y` 自动回答安装确认。
- `git --version` 查看 Git 是否安装及其版本。

### 6.2 GitHub Fine-grained Token（临时使用）

- [ ] 创建 Fine-grained personal access token。
- [ ] 仓库范围选择只允许访问目标仓库。
- [ ] 只申请当前任务需要的权限；如果要推送代码，通常需要目标仓库的 `Contents: Read and write`。
- [ ] 设置过期时间；用完后撤销，不要把 Token 写进脚本、仓库或聊天记录。

HTTPS 推送时，Git 提示输入密码的位置填写 Token。Token 相当于密码，要按秘密凭据保护。

```bash
git config --global credential.helper store
```

`credential.helper store` 会让 Git 记住凭据，但通常以明文保存到用户目录下的凭据文件。仅在可接受该风险的临时环境使用；长期服务器部署优先考虑最小权限的 Deploy Key 或 GitHub App。用完后可取消设置：

```bash
git config --global --unset credential.helper
```

### 6.3 Deploy Key（服务器只访问单个仓库）

服务器上为部署单独生成密钥：

```bash
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key -C "deploy-key"
cat ~/.ssh/deploy_key.pub
```

- `-t ed25519` 选择密钥算法。
- `-f` 指定密钥文件路径；不带 `.pub` 的是私钥，带 `.pub` 的是公钥。
- `-C` 写入备注。
- 把公钥添加到 GitHub 目标仓库的 Deploy keys 设置中。只需要拉取代码时，保持只读；只有确实需要推送时才开启写权限。

临时对一次 clone 命令指定私钥：

```bash
GIT_SSH_COMMAND='ssh -i ~/.ssh/deploy_key -o IdentitiesOnly=yes' git clone git@github.com:OWNER/REPO.git
```

- `GIT_SSH_COMMAND` 只影响这一次 Git 命令。
- `-i` 指定私钥。
- `-o IdentitiesOnly=yes` 让 SSH 只使用指定的身份密钥。
- 把 `OWNER/REPO` 换成 GitHub 用户名或组织名和仓库名。

只对当前仓库设置：

```bash
cd /opt/Auth
git config core.sshCommand 'ssh -i ~/.ssh/deploy_key -o IdentitiesOnly=yes'
```

在当前仓库取消：

```bash
git config --unset core.sshCommand
```

GitHub Deploy Key 绑定到单个仓库；私钥持有者可以获得该仓库授予的读写能力，所以服务器和私钥文件都要保护好。参考：[GitHub Deploy Keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)。

## 7. Docker 服务安装与日常操作

### 7.1 安装 Docker

Docker CE 的安装步骤会随发行版变化。Debian/Ubuntu 需要先配置 Docker 官方软件源，再安装 Docker Engine；不要假设单独运行 `apt-get install docker-ce` 在新服务器上一定可用。按系统选择官方步骤：[Docker Engine 安装文档](https://docs.docker.com/engine/install/)。

Docker 官方软件源配置好之后，Debian/Ubuntu 常见安装包为：

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- `docker-ce` 是 Docker Engine 服务端。
- `docker-ce-cli` 是命令行工具。
- `containerd.io` 是容器运行时组件。
- `docker-buildx-plugin` 提供 Buildx 构建功能。
- `docker-compose-plugin` 提供 `docker compose` 命令。

在适用 RPM 系统上使用 Docker 官方仓库提供的安装说明，不要直接假设 `dnf install docker` 安装的一定是 Docker CE；软件包名称和来源可能不同。

### 7.2 启动、停止、重启 Docker

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker --no-pager
sudo systemctl restart docker
sudo systemctl stop docker
```

- `start` 立即启动服务。
- `enable` 设置开机自动启动。
- `status` 查看服务状态。
- `restart` 重启服务；运行中的容器可能短暂中断，生产服务操作前先确认影响。
- `stop` 停止服务。

常见情况：

```bash
# 修改 Docker 的 systemd unit 文件后，重新读取 unit 定义，再重启服务
sudo systemctl daemon-reload
sudo systemctl restart docker

# 修改 Docker daemon 配置文件 /etc/docker/daemon.json 后，通常重启 Docker 即可
sudo systemctl restart docker
```

`daemon-reload` 是让 systemd 重新读取 unit 文件；只有修改了 systemd unit 或 drop-in 配置时才需要。`daemon-reexec` 会让 systemd 管理进程重新执行，主要用于调试或 systemd 升级等特殊情况，不是 Docker 的常规重启步骤。参考：[systemctl 手册](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)。

如果 Docker 是通过 Snap 安装的，Snap 服务命令可能是：

```bash
sudo snap services docker
sudo snap restart docker
```

只有 Snap 安装的 Docker 才使用 `snap restart`；先用 `snap list docker` 确认安装来源。

### 7.3 查看 Docker 状态、镜像和日志

```bash
docker version
sudo docker info
docker image ls
docker ps
docker ps -a
docker logs --tail 100 <容器名或ID>
```

- `docker version` 查看客户端和服务端版本。
- `docker info` 查看整体配置和运行情况，包括镜像数量、存储驱动、Docker 根目录等；它不只是查看镜像配置。
- `docker image ls` 列出本地镜像。
- `docker ps` 列出运行中的容器；`-a` 包括已经停止的容器。
- `docker logs` 查看容器日志；`--tail 100` 只显示最后 100 行。

查看 Docker 磁盘使用：

```bash
docker system df
```

### 7.4 拉取镜像、启动 Compose 服务

拉取镜像：

```bash
docker pull fredliang/derper
```

`docker pull` 从镜像仓库下载镜像；后面的内容是镜像名称。生产环境最好明确版本标签，例如 `fredliang/derper:版本号`，避免 `latest` 更新后产生意外变化。

在含 `compose.yaml` 或 `docker-compose.yml` 的项目目录启动服务：

```bash
docker compose up -d
docker compose ps
docker compose logs -f --tail 100
docker compose restart
docker compose down
```

- `up` 创建并启动服务。
- `-d` 在后台运行。
- `ps` 查看 Compose 服务状态。
- `logs -f` 持续查看日志；`--tail 100` 显示最近 100 行。
- `restart` 重启服务。
- `down` 停止并移除 Compose 创建的容器和网络；默认不删除命名卷。

### 7.5 清理 Docker 缓存和无用数据

先查看占用：

```bash
docker system df
```

清理构建缓存（范围较窄）：

```bash
docker builder prune
```

清理未使用的 Docker 数据：

```bash
docker system prune
docker system prune -a
```

- `docker system prune` 会删除停止的容器、未使用网络、悬空镜像和构建缓存。
- `-a` 还会删除所有没有被容器使用的镜像；之后可能需要重新下载。
- 默认不会删除卷。增加 `--volumes` 才会清理未使用的匿名卷；卷可能存有业务数据，不要在不确认数据用途时使用。
- 不要随意加 `-f` / `--force`：它会跳过确认提示。

Docker 官方说明：[docker system prune](https://docs.docker.com/reference/cli/docker/system/prune/)、[docker info](https://docs.docker.com/reference/cli/docker/system/info/)。

## 8. 路径和文件管理

项目和服务文件一般可以放在 `/opt`，例如 `/opt/Auth`。应用数据也可能放在 `/var/lib/<应用名>`；日志常见于 `/var/log` 或 systemd journal。具体位置以软件文档和部署配置为准。

```bash
pwd
ls -lah
cd /opt
mkdir -p /opt/Auth
```

- `pwd` 显示当前目录。
- `ls -lah` 列出文件；`-l` 长格式，`-a` 包含隐藏文件，`-h` 使用易读文件大小。
- `cd` 切换目录。
- `mkdir -p` 创建目录及缺失的父目录。

查看文件权限：

```bash
ls -l /path/to/file
stat /path/to/file
```

修改权限前先确认文件用途和所有者，不要用 `chmod -R 777` 作为通用修复办法。`777` 会让所有本机用户都能读、写、执行目标文件。

## 9. 一页式日常检查清单

- [ ] 登录后确认机器和用户：`hostname`、`whoami`。
- [ ] 看系统资源：`top`、`free -h`、`df -h`、`uptime`。
- [ ] 看监听端口和进程：`sudo ss -lntup`、`sudo lsof -i :<端口>`。
- [ ] 看最近成功和失败登录：`last -n 20`、`sudo lastb -n 20`、查看认证日志。
- [ ] 看关键服务状态：`sudo systemctl status <服务名> --no-pager`。
- [ ] 看最近错误：`sudo journalctl -p err --since "24 hours ago"`。
- [ ] 确认数据库等内部服务没有绑定到公网地址。
- [ ] 系统更新前看清将要升级、安装或移除的软件包。
- [ ] 删除容器、镜像、卷或日志前，确认它们是否还被业务使用，并确认有备份。
- [ ] 做完改动后重新检查服务状态、端口、日志，并从新 SSH 窗口测试登录。

## 10. 服务器安全加固检查清单

### 10.1 登录与账号安全

- [ ] SSH 只允许需要登录的普通用户；确认没有不认识的账号或离职用户账号。
- [ ] 确认 SSH 使用公钥登录，已关闭密码登录和 root 直接登录。
- [ ] 检查最近的失败登录记录；出现异常来源或频率时，继续核对成功登录记录和对应时间的系统日志。

```bash
sudo lastb -n 30
last -n 30
sudo journalctl -u ssh -u sshd --since "24 hours ago"
```

- [ ] 确认每个用户使用自己的密钥；离开项目或不再需要访问时，及时从 `authorized_keys` 删除对应公钥。
- [ ] 私钥只保存在需要使用它的电脑或服务器上，不通过聊天、邮件或代码仓库传递。
- [ ] sudo 权限只授予确实需要管理权限的账号；完成管理员操作后退出 `sudo -i` shell。

### 10.2 端口与公网暴露

- [ ] 检查所有监听端口，确认每个公网可访问的端口都有明确用途。

```bash
sudo ss -lntup
```

- [ ] 防火墙和云厂商安全组只放行需要的端口；SSH 端口变更后，确认新端口可用再关闭旧端口。
- [ ] 数据库、缓存、管理后台等内部服务不直接开放给整个公网；优先限制到 `127.0.0.1`、私有网络或指定来源 IP。
- [ ] 检查 Docker Compose 的 `ports` 配置，避免无意间把数据库或管理端口绑定到所有网卡。

```bash
sudo ss -lntup | grep -E ':(3306|5432|6379|27017)\b'
```

上面的例子检查 MySQL、PostgreSQL、Redis 和 MongoDB 的常见端口；如果实际服务使用其他端口，也要一并检查。Docker 发布端口时还要核对云安全组和 Docker 的防火墙规则。

- [ ] 不需要公网访问的服务，停止服务或删除对应的端口放行规则；不要只依赖“端口比较冷门”来保护服务。

### 10.3 系统和服务安全

- [ ] 定期安装系统和服务器软件的安全更新；升级前确认变更内容，并安排必要的重启。
- [ ] 移除不再使用的软件和服务，减少对外监听的程序。
- [ ] 检查 Nginx、数据库、Docker 等服务是否以专用普通账号运行，避免应用日常使用 root 权限。
- [ ] Docker 镜像使用可信来源和明确版本；更新镜像前了解变更，更新后检查容器状态和日志。
- [ ] 不随意运行来源不明的安装脚本；需要运行时先检查脚本内容和来源。

### 10.4 密钥、密码和数据保护

- [ ] 不把密码、GitHub Token、私钥或数据库凭据提交到 Git 仓库、公开日志或命令示例中。
- [ ] `.env` 等包含密钥的配置文件应限制读取权限，并确认已加入 `.gitignore`；检查是否曾经被提交到仓库。
- [ ] 凭据只授予完成任务所需的最小权限；不再使用或怀疑泄露时，及时撤销或轮换。
- [ ] 重要数据有独立备份；备份不只保存在同一台服务器上，并限制备份文件的访问权限。
- [ ] 定期试着从备份恢复关键数据，确认备份确实可用；没有验证过的备份不能视为可靠备份。

### 10.5 异常发现与应急

- [ ] 定期查看失败登录、成功登录、异常进程和监听端口；把异常时间、来源 IP 和相关日志记录下来。
- [ ] 发现疑似入侵时，先保留日志和必要证据；不要急着删除文件或清空日志。
- [ ] 怀疑密钥或 Token 泄露时，尽快撤销或更换凭据，并检查相关账号的登录和操作记录。
- [ ] 重要服务有可用的恢复办法：知道如何停止服务、恢复配置、恢复备份，以及如何联系云厂商处理网络或账号问题。

## 11. 参考资料

- [OpenSSH `sshd_config` 手册](https://man.openbsd.org/sshd_config.5)
- [Docker Engine 安装文档](https://docs.docker.com/engine/install/)
- [Docker `system prune` 文档](https://docs.docker.com/reference/cli/docker/system/prune/)
- [Docker `info` 文档](https://docs.docker.com/reference/cli/docker/system/info/)
- [systemd `systemctl` 手册](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [GitHub Fine-grained personal access token 文档](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [GitHub Deploy Key 文档](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)
- [Linux 登录日志实践文章（原笔记链接）](https://blog.csdn.net/weixin_42197110/article/details/116864024)
