# 📝 Obsidian + GitHub 自动同步安装部署总结

---

## 📋 完整安装部署步骤

---

### 🔽 第一步：安装 Obsidian (Debian/Ubuntu)

```bash
# 下载 deb 安装包（需要代理）
wget -O obsidian.deb https://github.com/obsidianmd/obsidian-releases/releases/download/v1.5.12/obsidian_1.5.12_amd64.deb

# 安装
export PATH=$PATH:/usr/local/sbin:/usr/sbin:/sbin
dpkg -i obsidian.deb

# 修复依赖
apt-get install -f -y
apt-get install -y libsecret-1-0
```

✅ **完成**：obsidian 安装到 `/usr/bin/obsidian`

---

### 📂 第二步：创建知识库（Vault）

```bash
# 创建知识库目录
mkdir -p /root/Documents/ObsidianVault
cd /root/Documents/ObsidianVault

# 初始化 Git
git init

# 创建 .gitignore 文件
cat > .gitignore << 'EOF'
# Obsidian 工作区和缓存
.obsidian/workspace
.obsidian/workspaces
.obsidian/cache
.obsidian/plugins/*/data.json
.obsidian/app.json
*.log

# 系统文件
.DS_Store
Thumbs.db

# 大文件（超过 100MB 的不要推 GitHub）
*.zip
*.tar.gz
*.rar
*.pdf
EOF
```

✅ **完成**：知识库目录创建好了，Git 初始化完成

---

### 📡 第三步：配置 Git 远程仓库

```bash
# 添加 GitHub 远程仓库（替换成你的仓库）
git remote add origin git@github.com:your-username/your-repo.git

# 配置 Git 用户信息
git config --global user.name "your-name"
git config --global user.email "your-email@example.com"

# 第一次提交
git add .
git commit -m "🎉 初始化 Obsidian 知识库"
```

✅ **完成**：本地准备好了

---

### 🔑 第四步：配置 SSH 密钥（用于 GitHub 认证）

```bash
# 生成 SSH 密钥（一路回车，不需要密码）
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -q

# 查看公钥（复制到 GitHub → Settings → SSH and GPG keys → New SSH key）
cat ~/.ssh/id_ed25519.pub
```

**GitHub 添加密钥后**，测试推送：

```bash
# 添加 GitHub 到 known_hosts
ssh-keyscan github.com >> ~/.ssh/known_hosts

# 推送第一次提交
git push -u origin master
```

✅ **完成**：GitHub 认证配置好了，可以推送了

---

### 🔄 第五步：配置每日自动同步

```bash
# 创建自动同步脚本
cat > /root/.obsidian-git-sync.sh << 'EOF'
#!/bin/bash
# Obsidian Git 自动同步脚本
# 每天自动同步一次

VAULT_DIR="/root/Documents/ObsidianVault"
REMOTE_NAME="origin"
BRANCH="master"

cd "$VAULT_DIR" || exit 1

# 使用代理
export http_proxy=socks5://127.0.0.1:1080
export https_proxy=socks5://127.0.0.1:1080
export ALL_PROxy=socks5://127.0.0.1:1080

echo "=== $(date) 开始同步 Obsidian ==="

# 拉取远程更改
git pull --rebase "$REMOTE_NAME" "$BRANCH"

# 添加所有本地更改
git add -A

# 提交（如果有更改）
git commit -m "🔄 自动同步 $(date +'%Y-%m-%d %H:%M')"

# 推送到 GitHub
git push "$REMOTE_NAME" "$BRANCH"

echo "=== $(date) 同步完成 ==="
EOF

# 添加执行权限
chmod +x /root/.obsidian-git-sync.sh

# 添加定时任务：每天凌晨 2 点自动同步
(crontab -l ; echo "0 2 * * * /root/.obsidian-git-sync.sh >> /var/log/obsidian-git-sync.log 2>&1") | crontab -
```

✅ **完成**：每天凌晨 2 点自动同步一次

---

## 📊 当前部署完成总结

| 项目 | 信息 |
|------|------|
| **Obsidian 版本** | 1.5.8 |
| **安装路径** | `/usr/bin/obsidian` |
| **知识库路径** | `/root/Documents/ObsidianVault/` |
| **GitHub 仓库** | `git@github.com:talang-tech/obsidian-cloud.git` |
| **自动同步** | 每天凌晨 **2 点** 自动同步 |
| **同步脚本** | `/root/.obsidian-git-sync.sh` |
| **同步日志** | `/var/log/obsidian-git-sync.log` |

---

## 🔧 日常使用维护

### 1️⃣ **本地使用（推荐）**

在你的 Windows/macOS/Linux 本地电脑：

```bash
# 克隆知识库
git clone git@github.com:talang-tech/obsidian-cloud.git ~/ObsidianVault

# 用 Obsidian 打开这个文件夹，开始写笔记
# 修改完之后：
git add .
git commit -m "update: xxx"
git push origin master
```

服务器会在每天凌晨自动 pull 你的更改，保持同步。

---

### 2️⃣ **手动触发同步（服务器上）**

如果需要马上同步，手动运行：

```bash
/root/.obsidian-git-sync.sh
```

---

### 3️⃣ **查看同步日志**

```bash
# 查看最近的日志
tail -f /var/log/obsidian-git-sync.log

# 查看完整日志
cat /var/log/obsidian-git-sync.log
```

---

### 4️⃣ **修改同步频率**

如果需要改变同步时间（比如每天同步多次），编辑 crontab：

```bash
crontab -e
```

**格式说明**：
```
# 分 时 日 月 周
# 每 6 小时同步一次：
0 */6 * * * /root/.obsidian-git-sync.sh >> /var/log/obsidian-git-sync.log 2>&1
```

---

### 5️⃣ **更新 Obsidian**

当有新版本时，重新下载安装：

```bash
# 查看新版本 https://github.com/obsidianmd/obsidian-releases/releases
wget -O obsidian-new.deb https://github.com/obsidianmd/obsidian-releases/download/vx.x.x/obsidian_vx.x.x_amd64.deb
dpkg -i obsidian-new.deb
```

---

### 6️⃣ **在服务器图形界面启动 Obsidian**

如果你有 VNC/rdp 连接服务器图形桌面：

```bash
# 需要加 --no-sandbox 参数以 root 运行
obsidian --no-sandbox &
```

---

## 🌿 最终架构

```
你的本地电脑 ← Git/GitHub ← 云服务器 ← 每日自动同步
     ↑            ↑             ↑
  本地写笔记      GitHub 备份       服务器自动备份
```

- 🖥️ **本地**：享受流畅的 Obsidian 体验
- ☁️ **云端**：自动备份，永不丢失
- 🔄 **双向同步**：本地更改 → GitHub → 服务器，服务器更改 → GitHub → 本地

---

*此文档由 OpenClaw 自动生成*
