# Obsidian + GitHub 自动同步 + V2Ray 代理 完整部署文档

---

## 概述

### 项目介绍
- **Obsidian**：本地Markdown笔记编辑器，基于文件存储，核心功能免费
- **GitHub 自动同步**：利用 Git 实现笔记云端备份和多设备同步，完全免费
- **V2Ray**：代理工具，帮助国内云服务器翻墙访问 GitHub 等网站

### 部署目标
- 在云服务器上部署 Obsidian 笔记知识库
- 配置 GitHub 云端自动备份，每天自动同步
- 配置 V2Ray 代理，让服务器能够访问 GitHub 被墙网站

### 适用环境
- **操作系统**：Debian / Ubuntu (x86_64)
- **硬件要求**：1核 2GB 以上即可
- **依赖**：需要有可用代理订阅节点，本文基于你的订阅自动配置

---

## 环境要求

| 项目 | 要求 |
|------|------|
| **操作系统** | Debian 18.04+ / Ubuntu 20.04+ |
| **架构** | x86_64 / amd64 |
| **网络 | 需要能够连接外网以安装，服务器在国内，需要订阅 V2Ray 代理 |
| **依赖** | `git` `wget` `dpkg` 系统自带，`proxychains4` 需要安装用于代理 |

---

## 安装步骤

---

### 1️⃣ 安装 V2Ray 代理

#### 1.1 下载安装 V2Ray

```bash
# 创建安装目录
mkdir -p /usr/local/v2ray && cd /usr/local/v2ray

# 下载（需要代理才能下载 GitHub 资源
proxychains4 wget https://github.com/v2fly/v2ray-core/releases/download/v5.22.0/v2ray-linux-64.zip

# 解压
unzip v2ray-linux-64.zip
chmod +x v2ray

# 创建软链接
ln -sf /usr/local/v2ray/v2ray /usr/local/bin/v2ray

# 复制 systemd 服务文件
cp systemd/system/v2ray.service /etc/systemd/system/
cp systemd/system/v2ray@.service /etc/systemd/system/
```

#### 1.2 配置 systemd 服务

```bash
# 重载 systemd
systemctl daemon-reload

# 开机自启
systemctl enable v2ray

# 启动服务
systemctl start v2ray
```

#### 1.3 验证安装完成后配置节点

> 你需要先有一个订阅链接，本文档假设你的订阅链接，替换下面脚本自动生成配置

```bash
# 下载解码订阅（替换成你的订阅链接
curl -s "https://your-subscription-link" | base64 -d > /path/to/nodes.txt

# 使用工具生成配置（工具脚本在 /root/.openclaw/workspace/ 已经准备好了）
python3 /root/.openclaw/workspace/generate_config.py <node-number>
systemctl restart v2ray
```

**默认代理信息**：
- SOCKS5 代理：`127.0.0.1:1080`
- HTTP 代理：`127.0.0.1:8080`

---

### 2️⃣ 安装 Obsidian

#### 2.1 下载安装 deb 包

```bash
# 下载（需要通过代理下载）
proxychains4 wget -O obsidian.deb https://github.com/obsidianmd/obsidian-releases/releases/download/v1.5.12/obsidian_1.5.12_amd64.deb

# 安装
export PATH=$PATH:/usr/local/sbin:/usr/sbin:/sbin
dpkg -i obsidian.deb

# 修复依赖
apt-get install -f -y
apt-get install -y libsecret-1-0
```

#### 2.2 验证安装

```bash
which obsidian
```

成功输出 `/usr/bin/obsidian` 说明安装成功。root 运行需要加 `--no-sandbox` 参数：

```bash
obsidian --no-sandbox
```

---

### 3️⃣ 创建 Obsidian 知识库

#### 3.1 创建知识库目录

```bash
# 创建知识库目录
mkdir -p /root/Documents/ObsidianVault
cd /root/Documents/ObsidianVault

# 初始化 Git
git init
```

#### 3.2 创建 .gitignore

```bash
cat > .gitignore << 'EOF'
# Obsidian 工作区缓存
.obsidian/workspace
.obsidian/workspaces
.obsidian/cache
.obsidian/plugins/*/data.json
.obsidian/app.json
*.log

# 系统文件
.DS_Store
Thumbs.db

# 大文件（超过 100MB 不推送 GitHub
*.zip
*.tar.gz
*.rar
*.pdf
EOF
```

#### 3.3 配置 Git 用户信息

```bash
# 替换成你的 GitHub 用户名和邮箱
git config --global user.name "your-name"
git config --global user.email "your-email@example.com"
```

#### 3.4 添加 GitHub 远程仓库

```bash
# 替换成你的 GitHub 仓库
git remote add origin git@github.com:your-username/your-repo.git
```

#### 3.5 初始提交

```bash
git add .
git commit -m "🎉 初始化 Obsidian 知识库"
```

---

### 4️⃣ 配置 SSH 密钥推送到 GitHub

#### 4.1 生成 SSH 密钥

```bash
# 一路回车，不需要密码
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -q
```

#### 4.2 查看公钥，复制添加到 GitHub

```bash
cat ~/.ssh/id_ed25519.pub
```

**添加步骤**：
1. GitHub → 头像 → Settings → SSH and GPG keys → New SSH key
2. Title 填 `obsidian-server`
3. 粘贴上面输出的公钥 → Add SSH key

#### 4.3 添加 GitHub 验证可以推送了

```bash
# 添加 GitHub 到 known_hosts
ssh-keyscan github.com >> ~/.ssh/known_hosts

# 推送第一次提交
export http_proxy=socks5://127.0.0.1:1080
export https_proxy=socks5://127.0.0.1:1080
git push -u origin master
```

---

### 5️⃣ 配置每日自动同步

#### 5.1 创建自动同步脚本

```bash
cat > /root/.obsidian-git-sync.sh << 'EOF'
#!/bin/bash
# Obsidian Git 自动同步脚本
# 每天自动同步一次

# 配置
VAULT_DIR="/root/Documents/ObsidianVault"
REMOTE="origin"
BRANCH="master"

# 进入仓库目录
cd "$VAULT_DIR" || exit 1

# 走代理访问 GitHub
export http_proxy=socks5://127.0.0.1:1080
export https_proxy=socks5://127.0.0.1:1080
export ALL_PROXY=socks5://127.0.0.1:1080

echo "=== \$(date) 开始同步 Obsidian ==="

# 拉取远程更改
git pull --rebase "$REMOTE" "$BRANCH"

# 添加所有更改
git add -A

# 提交更改（有更改才提交）
git commit -m "🔄 自动同步 \$(date +'%Y-%m-%d %H:%M')"

# 推送到 GitHub
git push "$REMOTE" "$BRANCH"

echo "=== \$(date) 同步完成 ==="
EOF
```

#### 增加执行权限

```bash
chmod +x /root/.obsidian-git-sync.sh
```

#### 5.2 添加定时任务（每天凌晨 2 点自动同步

```bash
# 添加定时任务
(crontab -l ; echo "0 2 * * * /root/.obsidian-git-sync.sh >> /var/log/obsidian-git-sync.log 2>&1") | crontab -
```

---

## ✅ 部署完成

---

## 验证安装成功

### 验证 V2Ray

```bash
# 检查服务状态
systemctl status v2ray

# 检查版本
v2ray version

# 测试配置
v2ray test -c /usr/local/etc/v2ray/config.json
```

**预期输出**：配置测试通过说明配置合法。

### 验证 Obsidian

```bash
which obsidian
```

**预期输出**：输出 `/usr/bin/obsidian` 说明安装成功。

### 验证 Git 同步

```bash
# 手动运行一次看看
/root/.obsidian-git-sync.sh
```

**预期输出**：正常推送成功说明配置正确。

---

## 使用方式

### V2Ray 代理使用

#### 命令行走代理

```bash
# 所有需要走代理的命令前面加 proxychains4
proxychains4 wget https://github.com/...
proxychains4 git clone https://github.com/...
```

#### 设置环境变量

```bash
# 当前终端生效
export http_proxy=socks5://127.0.0.1:1080
export https_proxy=socks5://127.0.0.1:1080
export ALL_PROXY=socks5://127.0.0.1:1080
```

#### Git 全局设置代理

```bash
git config --global http.proxy socks5://127.0.0.1:1080
git config --global https.proxy socks5://127.0.0.1:1080

# 取消代理
# git config --global --unset http.proxy
# git config --global --unset https.proxy
```

### Obsidian 使用

#### 本地使用推荐（体验最好）

在你的**本地电脑**上：

```bash
# 克隆知识库
git clone git@github.com:your-username/your-repo.git ~/ObsidianVault

# 用本地 Obsidian 打开这个文件夹，开始写笔记
```

本地修改后：

```bash
git add .
git commit -m "update: xxx"
git push origin master
```

服务器会每天自动拉取你的更改。

#### 服务器图形界面启动

如果你有 VNC/rdp 连接到服务器图形桌面：

```bash
obsidian --no-sandbox &
```

---

## 维护指南

### V2Ray 维护

| 操作 | 命令
---|---
| 查看状态 | `systemctl status v2ray
| 启动服务 | `systemctl start v2ray`
| 停止服务 | `systemctl stop v2ray`
| 重启服务 | `systemctl restart v2ray`
| 查看日志 | `journalctl -u v2ray -f`
| 重载配置 | `vim /usr/local/etc/v2ray/config.json`

### Obsidian Git 同步维护

| 操作 | 命令
---|---
| 手动立即同步 | `/root/.obsidian-git-sync.sh
| 查看同步日志 | `tail -f /var/log/obsidian-git-sync.log`
| 编辑定时任务 | `crontab -e`
| 查看同步频率 | 默认每天凌晨 **每天凌晨 2 点自动同步，可以修改定时任务改变频率

### 更新升级

#### 更新 Obsidian

```bash
# 下载新版 deb
proxychains4 wget -O obsidian-new.deb https://github.com/obsidianmd/obsidian-releases/releases/download/vx.x.x/obsidian_vx.x.x_amd64.deb
dpkg -i obsidian-new.deb
```

#### 更新 V2Ray

```bash
cd /usr/local/v2ray
proxychains4 wget https://github.com/v2fly/v2ray-core/releases/download/v-new-version/v2ray-linux-64.zip
unzip -o v2ray-linux-64.zip
systemctl restart v2ray
```

### 备份

- 整个知识库备份，直接 git push 到 GitHub 就是备份
- 克隆就是恢复，git clone 仓库就恢复了

---

### 故障排查

#### 推送失败 🔍

**问题**：`fatal: could not read Username for 'https://github.com': No such device or address

**解决**：使用 SSH 方式拉推，已经配置了 SSH 密钥，确保你的 GitHub 已经添加公钥就不会有这个问题。

**问题**：连接超时，SSL 错误

**解决**：一般是网络问题，检查代理是否正常，换节点试试。

**问题**：checksum 错误

**解决**：不影响，继续就行。

---

## 最终目录结构

```
/root/Documents/ObsidianVault/  → 你的笔记知识库，自动同步
/usr/local/v2ray/ → V2Ray 安装目录
/etc/systemd/system/v2ray.service → 系统服务文件
/root/.obsidian-git-sync.sh → 自动同步脚本
/var/log/obsidian-git-sync.log → 同步日志
/root/Documents/ObsidianVault/OB_INSTALL_SUMMARY.md → 本文档
/root/Documents/ObsidianVault/V2RAY_DEPLOY_SUMMARY.md → V2Ray 总结文档
```

---

*本文档由 OpenClaw 自动生成*
