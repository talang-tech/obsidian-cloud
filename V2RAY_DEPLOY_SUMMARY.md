# 📝 V2Ray 代理安装部署总结

---

## 📋 部署信息

| 项目 | 配置 |
|------|------|
| **V2Ray 版本** | 5.22.0 (V2Fly 社区版) |
| **安装路径** | `/usr/local/v2ray/` |
| **可执行文件** | `/usr/local/bin/v2ray` |
| **配置文件** | `/usr/local/etc/v2ray/config.json` |
| **系统服务** | `/etc/systemd/system/v2ray.service` |
| **SOCKS5 代理端口** | `127.0.0.1:1080` |
| **HTTP 代理端口** | `127.0.0.1:8080` |
| **当前节点** | 🇺🇸 UnitedStates1 (node 47) vmess |
| **订阅地址** | `https://prp.zz01.eu.org/scrb/8591b0e2628c018abae09323e238f6e1` |

---

## 🔧 安装步骤

### 1️⃣ 下载解压安装

```bash
# 创建目录
mkdir -p /usr/local/v2ray && cd /usr/local/v2ray

# 下载 V2Ray 压缩包（通过代理）
proxychains4 wget https://github.com/v2fly/v2ray-core/releases/download/v5.22.0/v2ray-linux-64.zip

# 解压
unzip v2ray-linux-64.zip
chmod +x v2ray

# 创建软链接
ln -sf /usr/local/v2ray/v2ray /usr/local/bin/v2ray

# 安装 systemd 服务
cp systemd/system/v2ray.service /etc/systemd/system/
cp systemd/system/v2ray@.service /etc/systemd/system/
```

### 2️⃣ 配置系统服务

```bash
# 重载 systemd
systemctl daemon-reload

# 启用开机自启
systemctl enable v2ray

# 启动服务
systemctl start v2ray

# 检查状态
systemctl status v2ray
```

### 3️⃣ 配置节点

使用订阅链接获取节点列表，选择节点生成配置：

**当前配置结构：**
- 订阅链接解码后得到 68 个节点
- 按地区分类：瑞典 8、越南 8、加拿大 7、新加坡 8、日本 7、香港 8、台湾 8、美国+德国 14
- 支持 vmess/vless/hysteria2 多种协议
- 国内服务器使用海外节点，通过代理访问 GitHub 等网站

---

## 🚀 使用方法

### 命令行工具通过代理

```bash
# 使用 proxychains4 让命令走代理
proxychains4 wget https://github.com/...
proxychains4 git clone https://github.com/...
proxychains4 curl https://...
```

### 配置环境变量（当前终端生效）

```bash
# SOCKS5
export http_proxy=socks5://127.0.0.1:1080
export https_proxy=socks5://127.0.0.1:1080
export ALL_PROXY=socks5://127.0.0.1:1080
```

### Git 配置代理

```bash
# 全局配置
git config --global http.proxy socks5://127.0.0.1:1080
git config --global https.proxy socks5://127.0.0.1:1080

# 取消全局配置
# git config --global --unset http.proxy
# git config --global --unset https.proxy
```

---

## 🔄 切换节点

### 1️⃣ 查看所有节点

```bash
# 完整节点列表在本地
cat /root/.openclaw/workspace/nodes_list.txt
```

### 2️⃣ 切换节点命令

```bash
# 使用工具切换节点（节点编号 1-68）
python3 /root/.openclaw/workspace/generate_config.py <node-number>
systemctl restart v2ray
```

**示例：切换到香港节点 32**
```bash
python3 /root/.openclaw/workspace/generate_config.py 32
systemctl restart v2ray
```

### 3️⃣ 更新订阅

如果订阅链接更新了，重新获取：

```bash
# 重新下载解码订阅
curl -s "https://prp.zz01.eu.org/scrb/8591b0e2628c018abae09323e238f6e1" | base64 -d > /root/.openclaw/workspace/v2ray_config_decoded.json

# 重新生成节点列表
python3 /root/.openclaw/workspace/list_nodes.py > /root/.openclaw/workspace/nodes_list.txt
```

---

## 🛠️ 维护管理

### 检查状态

```bash
# 查看服务状态
systemctl status v2ray

# 查看版本
v2ray version

# 测试配置
v2ray test -c /usr/local/etc/v2ray/config.json

# 查看日志
journalctl -u v2ray -f
```

### 重启服务

```bash
systemctl restart v2ray
```

### 停止服务

```bash
systemctl stop v2ray
```

### 测试代理

```bash
# 测试能否访问 GitHub
proxychains4 curl -I https://github.com
```

---

## 📊 节点列表（按地区）

| 地区 | 节点编号 | 协议 | 倍率 |
|------|---------|------|------|
| 🇸🇪 瑞典 | 1-8 | vmess/hysteria2 | 0.8 |
| 🇻🇳 越南 | 9-16 | vless/hysteria2 | 0.8 |
| 🇨🇦 加拿大 | 17-23 | vless/hysteria2 | 1.0 |
| 🇸🇬 新加坡 | 24-31 | vless/hysteria2 | 1.0 |
| 🇯🇵 日本 | 40-46 | vmess/hysteria2 | 1.0 |
| 🇭🇰 香港 | 32-39 | vless/hysteria2 | 1.5 |
| 🇹🇼 台湾 | 47-62 | vless/hysteria2 | 2.5 |
| 🇺🇸 🇩🇪 美国/德国 | 47-68 | vmess/vless | 1.0-1.0 |

---

## 💡 使用建议

- **服务器在国内** → 使用海外节点可以正常访问 GitHub 等被墙网站
- **速度** → 距离越近延迟越低，香港/新加坡/台湾速度通常比美国快
- **切换节点** → 如果当前节点速度慢/连不上，可以切到其他地区节点试试
- **proxychains** → 给命令行工具提供代理支持，任何命令都可以通过 `proxychains4 command` 走代理

---

*此文档由 OpenClaw 自动生成*
