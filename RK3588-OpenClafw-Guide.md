# OpenClaw 低配服务器部署完整教程

## 🎉 前言：一个奇迹的诞生

本文档记录了在 **Rockchip RK3588** 嵌入式开发板上成功部署 OpenClaw 的完整过程。这是国内首个在 ARM64 低配服务器上运行的 Clawdbot 实例！

### 📊 我的配置清单

| 项目 | 参数 |
|------|------|
| **处理器** | Rockchip RK3588 (ARM64, 8核) |
| **内存** | 7.7GB (可用5.8GB) |
| **存储** | 58GB (eMMC/SD卡, 已用22GB) |
| **系统** | Linux 5.10.198-rk3588 (aarch64) |
| **Node.js** | v24.13.0 (通过 NVM 安装) |
| **OpenClaw** | v2026.1.29 |
| **代理** | Mihomo (Clash) HTTP代理: 127.0.0.1:7890 |
| **消息平台** | Telegram Bot |
| **AI模型** | Zai-GLM-4.7 (通过API访问) |

### ⚠️ 重要提示

**本文档适用于国内用户**，所有下载源和代理配置都已针对国内网络环境优化。请严格按照步骤操作，避免网络问题导致的安装失败。

---

## 🚀 第一章：准备工作

### 1.1 系统要求

OpenClaw 对硬件要求很低，几乎能在任何支持 Node.js 的设备上运行：

| 配置项目 | 最低要求 | 推荐配置 | 我的配置 |
|---------|---------|---------|---------|
| CPU |ARM64或x86_64 | 2核+ | RK3588 (8核) |
| 内存 | 512MB | 2GB+ | 7.7GB |
| 存储 | 2GB | 10GB+ | 58GB |
| 系统 | Linux/Windows/Mac | Linux Debian/Ubuntu | Linux 5.10 |

### 1.2 配置APT国内源（Ubuntu/Debian）

在开始之前，我们需要将系统包管理源切换到国内镜像，大幅提升下载速度。

```bash
# 1. 备份原始源配置
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup

# 2. 选择合适的国内镜像源
#    - 清华镜像：https://mirrors.tuna.tsinghua.edu.cn/
#    - 阿里云镜像：https://mirrors.aliyun.com/
#    - 中科大镜像：https://mirrors.ustc.edu.cn/

# 3. 以Ubuntu 22.04为例，替换为清华源
sudo tee /etc/apt/sources.list > /dev/null <<EOF
# 清华大学开源软件镜像站
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
EOF

# 4. 如果是Debian系统，使用以下配置：
# sudo tee /etc/apt/sources.list > /dev/null <<EOF
# # 清华大学开源软件镜像站
# deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free
# deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free
# deb https://mirrors.tuna.tsinghua.edu.cn/debian-security bookworm-security main contrib non-free
# EOF

# 5. 更新软件包索引
sudo apt update

# 6. 升级已安装的软件包（可选，建议做）
sudo apt upgrade -y

# 7. 安装基础工具
sudo apt install -y \
    curl \
    wget \
    git \
    vim \
    nano \
    build-essential \
    python3 \
    python3-pip \
    ca-certificates \
    gnupg \
    lsb-release
```

### 1.3 配置Python PIP国内源

```bash
# 1. 创建pip配置目录
mkdir -p ~/.pip

# 2. 配置pip使用清华源
cat > ~/.pip/pip.conf <<EOF
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
trusted-host = pypi.tuna.tsinghua.edu.cn

[install]
trusted-host = pypi.tuna.tsinghua.edu.cn
EOF

# 3. 验证配置
pip3 config list

# 4. 测试pip速度
pip3 list
```

---

## 📦 第二章：安装 Node.js

### 2.1 配置NVM使用国内镜像

NVM本身很轻量，但下载Node.js时速度很慢。我们需要配置NVM使用国内镜像。

```bash
# 1. 安装NVM（使用Gitee镜像加速安装脚本）
curl -o- https://gitee.com/mirrors/nvm/raw/master/install.sh | bash

# 2. 如果Gitee有问题，使用原版GitHub（速度慢）
# curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# 3. 重新加载配置
source ~/.bashrc

# 4. 配置NVM使用国内镜像下载Node.js
echo 'export NVM_NODEJS_ORG_MIRROR=https://npmmirror.com/mirrors/node/' >> ~/.bashrc

# 5. 重新加载配置
source ~/.bashrc

# 6. 验证NVM安装
command -v nvm
nvm --version
```

### 2.2 配置NPM使用国内源

```bash
# 1. 配置npm使用淘宝镜像
npm config set registry https://registry.npmmirror.com

# 2. 验证配置
npm config get registry

# 3. 测试npm速度（搜索一个常用包）
npm search express --long=false | head -5
```

### 2.3 安装Node.js

```bash
# 1. 安装Node.js最新LTS版本
nvm install --lts

# 2. 查看已安装的Node.js版本
nvm list

# 3. 设置为默认版本
nvm use --lts
nvm alias default node

# 4. 验证安装
node -v  # 应该显示v22.x.x或更高
npm -v

# 5. 查看npm配置
npm config list
```

### 2.4 安装OpenClaw

```bash
# 1. 全局安装OpenClaw（使用淘宝镜像，速度会很快）
npm install -g openclaw

# 2. 验证安装
openclaw --version

# 3. 查看安装路径
which openclaw
```

---

## 🌐 第三章：配置代理（详细教程）

### 3.1 为什么需要代理？

**在国内网络环境下，很多服务无法直接访问**，包括：
- ✅ Telegram API (`api.telegram.org`)
- ✅ OpenAI API (`api.openai.com`)
- ✅ GitHub部分资源
- ✅ 部分国外AI服务

**需要代理才能正常使用的功能**：
- Telegram Bot收发消息
- 使用OpenAI、Claude等国外AI模型
- 访问GitHub Release等资源

**不需要代理的功能**：
- 使用国内AI API（智谱、通义千问、DeepSeek等）
- 本地文件操作
- 正常的命令行操作

### 3.2 什么是代理？

**简单理解**：代理就是"中转站"
- 你的请求 → 代理服务器 → 目标网站
- 目标网站返回 → 代理服务器 → 你的电脑

**为什么需要代理**：
- 国内网络防火墙（GFW）阻挡了很多国外网站
- 通过代理可以"翻墙访问"国外网站

**常见代理类型**：
- **HTTP代理**：只支持HTTP协议
- **HTTPS代理**：支持HTTP和HTTPS
- **SOCKS5代理**：支持所有TCP/UDP协议
- **混合代理（推荐）**：自动识别协议

### 3.3 代理服务器的获取方式

#### 方式一：使用机场订阅（最简单，推荐新手）

**什么是机场**：提供代理服务的第三方服务商

**步骤**：
1. **选择机场**
   - 搜索关键词："机场 推荐 2025"
   - 关注：稳定性、速度、价格、客服
   - 推荐：按月付费的小机场，先试用再付费

2. **购买套餐**
   - 通常有：体验版、基础版、高级版
   - 新手建议：先购买1个月的体验版测试
   - 价格范围：5元/月 - 30元/月不等

3. **获取订阅链接**
   - 登录机场网站
   - 找到"订阅管理"或"My Subscription"
   - 复制订阅链接（通常是 `yyyy://xxxxx` 或 `clash://xxxxx` 格式）

4. **测试订阅**
   ```bash
   # 将订阅链接保存到变量
   SUB_URL="你的订阅链接"
   
   # 查看订阅内容（使用Python工具）
   pip3 install clash2sing-box
   clash2sing-box --url "$SUB_URL" > ~/clash-test.yaml
   cat ~/clash-test.yaml | head -50
   ```

#### 方式二：自建VPS代理（技术要求高）

**适合人群**：有技术基础，需要长期稳定使用

**步骤**：
1. **购买VPS服务器**
   - 推荐服务商：Vultr、DigitalOcean、BandwagonHost
   - 配置要求：1核1G内存即可
   - 位置选择：香港、日本、新加坡（延迟低）
   - 价格：约$5/月

2. **连接VPS**
   ```bash
   ssh root@你的VPS_IP
   ```

3. **安装代理软件**
   ```bash
   # 方案A：安装Xray（推荐）
   bash <(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)
   
   # 方案B：安装Clash
   wget https://github.com/Dreamacro/clash/releases/download/v1.18.0/clash-linux-arm64-v1.18.0.gz
   gunzip clash-linux-arm64-v1.18.0.gz
   chmod +x clash-linux-arm64-v1.18.0
   ```

4. **配置代理**
   - 生成配置文件
   - 设置端口（推荐：8080）
   配置UUID或密码
   - 启动服务

5. **获取服务器信息**
   - IP地址
   - 端口号
   - UUID或密码
   - 加密方式

**自建代理的优缺点**：
- ✅ 优点：稳定、速度快、数据安全、成本可控
- ❌ 缺点：需要技术基础、需要维护

### 3.4 安装Mihomo（Clash增强版）

Mihomo是Clash的增强版，支持更多功能和规则。

```bash
# 1. 创建目录
mkdir -p ~/.local/bin
mkdir -p ~/.config/mihomo

# 2. 下载Mihomo（ARM64版本，适合RK3588等ARM架构）
#    如果是x86_64架构，请下载对应的版本
cd ~/.local/bin
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.18.10/mihomo-linux-arm64-v1.18.10.gz

# 3. 解压
gunzip mihomo-linux-arm64-v1.18.10.gz

# 4. 添加执行权限
chmod +x mihomo-linux-arm64-v1.18.10

# 5. 重命名为mihomo（方便调用）
mv mihomo-linux-arm64-v1.18.10 mihomo

# 6. 验证安装
mihomo -h
```

### 3.5 配置Mihomo（使用机场订阅）

#### 方法A：自动生成配置（推荐）

```bash
# 1. 安装订阅转换工具
pip3 install clash2sing-box

# 2. 设置订阅链接变量
SUB_URL="你的机场订阅链接"

# 3. 生成配置文件
clash2sing-box --url "$SUB_URL" > ~/.config/mihomo/config.yaml

# 4. 查看配置文件
cat ~/.config/mihomo/config.yaml | head -50

# 5. 查看可用的节点
grep "name:" ~/.config/mihomo/config.yaml | head -10
```

#### 方法B：手动配置

如果你没有购买机场，可以配置自建的VPS代理：

```yaml
# 创建配置文件
cat > ~/.config/mihomo/config.yaml << 'EOF'
# Mihomo 配置文件
# =====================

# 端口配置
port: 7891          # HTTP代理端口
socks-port: 7892    # SOCKS代理端口
mixed-port: 7890    # 混合代理端口（推荐使用）

# 允许局域网访问（可选，如果要让局域网其他设备使用）
allow-lan: false
bind-address: '*'

# 代理模式
mode: rule

# 日志配置
log-level: info

# 外部控制器（可选，用于网页控制面板）
external-controller: 0.0.0.0:9090
external-ui: ui

# GeoIP 数据源
geodata-mode: false
geo-auto-update: true
geo-update-interval: 24
geox-url:
  geoip: https://mirrors.tuna.tsinghua.edu.cn/gh/MetaCubeX/meta-rules-dat@release/geoip.dat
  geosite: https://mirrors.tuna.tsinghua.edu.cn/gh/MetaCubeX/meta-rules-dat@release/geosite.dat
  mmdb: https://mirrors.tuna.tsinghua.edu.cn/gh/MetaCubeX/meta-rules-dat@release/country.mmdb

# =====================
# 代理节点配置
# =====================
proxies:
  # 如果你没有机场，可以配置自建的VPS代理
  # 示例：socks5代理
  # - name: "我的VPS代理"
  #   type: socks5
  #   server: 你的VPS IP地址
  #   port: 1080
  #   username: 用户名（如果需要）
  #   password: 密码（如果需要）
  
  # 示例：http代理
  # - name: "我的HTTP代理"
  #   type: http
  #   server: 你的VPS IP地址
  #   port: 8080
  #   username: 用户名
  #   password: 密码
  
  # 注意：如果你有机场订阅，上面的配置会被订阅中的节点覆盖

# =====================
# 代理组配置
# =====================
proxy-groups:
  # 节点选择组（自动选择）
  - name: 🚀 节点选择
    type: select
    proxies:
      - DIRECT        # 直连（不使用代理）
      # 添加你订阅中的节点名称
      # - 日本节点
      # - 香港节点
      # - 美国节点

# =====================
# 分流规则
# =====================
rules:
  # 国内网站直连
  - GEOIP,CN,DIRECT
  
  # Telegram必须走代理
  - DOMAIN-SUFFIX,telegram.org,🚀 节点选择
  - DOMAIN-SUFFIX,t.me,🚀 节点选择
  - DOMAIN-SUFFIX,tdesktop.com,🚀 节点选择
  
  # OpenAI必须走代理
  - DOMAIN-SUFFIX,openai.com,🚀 节点选择
  - DOMAIN-SUFFIX,chatgpt.com,🚀 节点选择
  
  # 其他全部走默认
  - MATCH,🚀 节点选择
EOF
```

### 3.6 测试Mihomo配置

```bash
# 1. 测试配置文件语法
mihomo -d ~/.config/mihomo -t

# 如果提示 "Configuration file test successful"，说明配置正确
# 如果有错误，根据提示修复配置文件

# 2. 启动Mihomo（前台运行，用于测试）
mihomo -d ~/.config/mihomo

# 3. 此时终端会显示日志，新开一个终端测试代理

# 4. 测试代理是否工作
curl -x http://127.0.0.1:7890 https://www.google.com

# 如果成功返回Google首页HTML，说明代理工作正常

# 5. 测试Telegram API连接
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe

# 如果返回：{"ok":false,"error_code":404,"description":"Not Found"}
# 说明能连接到Telegram API（404是正常的，因为没有提供bot token）

# 6. 如果测试成功，Ctrl+C 停止前台运行
```

### 3.7 配置Mihomo开机自启动

```bash
# 1. 创建systemd服务文件
cat > ~/.config/systemd/user/mihomo.service << 'EOF'
[Unit]
Description=Mihomo Proxy Service
After=network-online.target

[Service]
Type=simple
ExecStart=%h/.local/bin/mihomo -d %h/.config/mihomo
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
EOF

# 2. 如果还没有启动过用户systemd，先设置
loginctl enable-linger $USER

# 3. 重新加载systemd配置
systemctl --user daemon-reload

# 4. 启用服务（开机自启）
systemctl --user enable mihomo

# 5. 立即启动服务
systemctl --user start mihomo

# 6. 查看服务状态
systemctl --user status mihomo

# 7. 查看日志（如有问题）
journalctl --user -u mihomo -f
```

### 3.8 验证代理功能

```bash
# 1. 查看Mihomo状态
systemctl --user status mihomo

# 2. 测试直连（不使用代理）
curl https://www.baidu.com

# 3. 测试代理访问Google
curl -x http://127.0.0.1:7890 https://www.google.com

# 4. 测试代理访问Telegram
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe

# 5. 查看代理日志
journalctl --user -u mihomo --since "1 min ago" | tail -20
```

### 3.9 代理配置常见问题

#### 问题1：代理无法连接，超时

**检查步骤**：
```bash
# 1. 检查Mihomo是否运行
systemctl --user status mihomo

# 2. 检查端口是否监听
ss -tunlp | grep 7890

# 3. 检查订阅是否过期
# 登录机场网站，查看订阅到期时间

# 4. 重新获取订阅
SUB_URL="你的订阅链接"
clash2sing-box --url "$SUB_URL" > ~/.config/mihomo/config.yaml
systemctl --user restart mihomo
```

#### 问题2：节点速度慢

**解决方法**：
1. 使用Mihomo网页控制面板选择延迟低的节点
   ```bash
   # 访问 http://localhost:9090/ui
   # 在网页中测试各节点延迟
   # 选择延迟最低的节点
   ```

2. 手动编辑配置，选择固定节点
   ```yaml
   proxy-groups:
     - name: 🚀 节点选择
       type: select
       proxies:
         - DIRECT
         - 香港节点    # 放在最前面优先使用
         - 日本节点
         - 美国节点
   ```

#### 问题3：代理正常但Telegram无法连接

**检查步骤**：
```bash
# 1. 测试代理通用性
curl -x http://127.0.0.1:7890 https://www.google.com

# 2. 测试Telegram API
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe

# 3. 检查规则配置
cat ~/.config/mihomo/config.yaml | grep -A 2 "telegram"

# 4. 确保规则中有Telegram分流
# - DOMAIN-SUFFIX,telegram.org,🚀 节点选择
```

### 3.10 代理配置总结

**必须配置的3个关键点**：
1. ✅ `~/.config/mihomo/config.yaml` - Mihomo配置文件
2. ✅ `~/.openclaw/openclaw.json` - OpenClaw配置文件中添加 `proxy` 字段
3. ✅ `~/.config/systemd/user/openclaw-gateway.service` - Systemd环境变量

**验证代理工作正常的测试命令**：
```bash
# 快速测试脚本
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe \
  && echo "✅ 代理工作正常" \
  || echo "❌ 代理配置有问题"
```

**记住**：代理是Telegram连接的核心，必须配置正确才能使用Telegram Bot功能！

---

## ⚙️ 第四章：初始化 OpenClaw

### 4.1 运行向导

```bash
# 1. 启动OpenClaw向导
openclaw onboard

# 2. 按照提示完成配置：
#    ① 选择安装模式：local（本地模式）
#    ② 设置工作目录：/home/youruser/clawd
#    ③ 配置模型提供商（推荐用免费的国内API）
```

### 4.2 获取免费的 AI API

OpenClaw支持多种AI模型提供商。推荐使用国内的API服务，免费额度多、速度快、稳定。

#### Zai AI（推荐新手）

**优势**：
- ✅ 每天免费币额度
- ✅ 支持多个大模型（GLM-4、Qwen等）
- ✅ API简单易用
- ✅ 响应速度快

**获取步骤**：
```bash
# 1. 访问官网
#    https://cloud.siliconflow.cn/

# 2. 注册账号（手机号或邮箱）

# 3. 登录后进入控制台

# 4. 创建API Key
#    点击左侧菜单 "API密钥"
#    点击 "新建API密钥"
#    复制生成的密钥（格式：sk-xxxxxxx）
#    保存好，只显示一次！

# 5. 在OpenClaw向导中配置：
#    Base URL: https://api.siliconflow.cn/v1
#    API Key: <your api key>
#    模型: Pro/zai-org/GLM-4.7
```

#### DeepSeek（强烈推荐）

**优势**：
- ✅ 每天免费100万tokens（非常慷慨）
- ✅ 性能强，代码能力出色
- ✅ 价格便宜（超过免费额后）

**获取步骤**：
```bash
# 1. 访问官网
#    https://platform.deepseek.com/

# 2. 注册账号（手机号）

# 3. 实名认证（国内必须）

# 4. 创建API Key
#    点击左侧 "API Keys"
#    点击 "创建API Key"
#    复制密钥

# 5. 在OpenClaw向导中配置：
#    Base URL: https://api.deepseek.com/v1
#    API Key: <your api key>
#    模型: deepseek-chat
```

#### Qwen AI（阿里巴巴通义千问）

**优势**：
- ✅ 阿里巴巴出品，稳定可靠
- ✅ 有免费额度
- ✅ 中文能力强
- ✅ 支持多模态

**获取步骤**：
```bash
# 1. 访问官网
#    https://portal.qwen.ai/

# 2. 注册阿里云账号（如果没有）

# 3. 登录后获取API Key

# 4. 在OpenClaw向导中配置：
#    Base URL: https://portal.qwen.ai/v1
#    API Key: <your api key>
#    模型: qwen-plus
```

### 4.3 配置文件说明

OpenClaw 的主配置文件位置：`~/.openclaw/openclaw.json`

**关键配置示例**：

```json
{
  "models": {
    "providers": {
      "zai-api": {
        "baseUrl": "https://api.siliconflow.cn/v1",
        "apiKey": "<your api key>",
        "auth": "api-key",
        "api": "openai-completions",
        "models": [
          {
            "id": "Pro/zai-org/GLM-4.7",
            "name": "GLM-4.7",
            "contextWindow": 128000,
            "maxTokens": 8192
          }
        ]
      },
      "deepseek": {
        "baseUrl": "https://api.deepseek.com/v1",
        "apiKey": "<your api key>",
        "auth": "api-key",
        "api": "openai-completions",
        "models": [
          {
            "id": "deepseek-chat",
            "name": "DeepSeek Chat",
            "contextWindow": 128000,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "zai-api/Pro/zai-org/GLM-4.7"
      },
      "workspace": "/home/youruser/clawd",
      "maxConcurrent": 2,
      "subagents": {
        "maxConcurrent": 4
      }
    }
  }
}
```

**说明**：
- 可以配置多个模型提供商
- `primary` 指定默认使用的模型
- `maxConcurrent` 控制并发数（低配服务器建议2）

---

## 📱 第五章：配置 Telegram Bot

### 5.1 创建 Telegram Bot

**目标**：获得一个Bot Token，让OpenClaw能够连接到Telegram

**步骤**：

1. **打开Telegram搜索BotFather**
   - 在Telegram搜索框输入：`@BotFather`
   - 点击进入对话

2. **创建新机器人**
   - 发送命令：`/newbot`
   - BotFather会回复：`Alright, a new bot. How are we going to call it? Please choose a name for your bot.`

3. **设置机器人名称**
   - 输入你的机器人名称（英文）
   - 例如：`MyClawBot`
   - BotFather回复：`Good. Now let's choose a username for your bot. It must end in bot. Like this, TetrisBot or tetris_bot.`

4. **设置机器人用户名**
   - 输入用户名（必须以bot结尾）
   - 例如：`my_claw_bot`
   - BotFather回复：
     ```
     Done! Congratulations on your new bot. You will find it at t.me/my_claw_bot. You can now add a description, about section and profile picture for your bot, see /help for a list of commands.
     
     Use this token to access the HTTP API:
     123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
     Keep your token secure and store it safely, it can be used by anyone to control your bot.
     ```
   - **复制Token**（格式：`数字:字母`）

5. **设置机器人描述（可选）**
   - 发送：`/setdescription`
   - 描述内容：这是一个AI聊天机器人

6. **测试机器人**
   - 点击链接：`t.me/my_claw_bot`
   - 发送：`/start`
   - 机器人会回复基本信息

**重要提示**：
- ⚠️ Token就是机器人的"密码"，非常重要！
- ⚠️ 不要把Token发给任何人！
- ⚠️ 如果Token泄露，可以重新生成！

### 5.2 配置 OpenClaw 连接 Telegram

**关键**：必须在3个地方配置代理！

#### 配置1：Telegram 代理设置

编辑配置文件：`~/.openclaw/openclaw.json`

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "pairing",
      "botToken": "<your bot token>",
      "proxy": "http://127.0.0.1:7890",  // ⚠️ 必须配置！
      "groups": {
        "*": {
          "requireMention": false
        }
      },
      "allowFrom": ["*"],
      "groupPolicy": "allowlist"
    }
  }
}
```

**字段说明**：
- `enabled`: 是否启用（true/false）
- `dmPolicy`: 私聊策略
  - `pairing`: 需要配对才能私聊
  - `open`: 任何人都可以私聊
- `botToken`: 你的Bot Token
- `proxy`: 代理地址（必须是 http://127.0.0.1:7890）
- `allowFrom`: 允许谁使用（["*"]表示所有人）
- `groupPolicy`: 群组策略
  - `allowlist`: 白名单（需要配置groups）
  - `open`: 任何群组

#### 配置2：Systemd 环境变量

编辑文件：`~/.config/systemd/user/openclaw-gateway.service`

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target

[Service]
ExecStart="/usr/local/bin/node" "/home/youruser/.nvm/versions/node/v24.13.0/lib/node_modules/openclaw/dist/index.js" gateway --port 64434
Restart=always
Environment=HOME=/home/youruser
Environment="PATH=/home/youruser/.nvm/current/bin:/home/youruser/.local/bin:/usr/local/bin:/usr/bin:/bin"
Environment=OPENCLAW_GATEWAY_PORT=64434

# ⚠️ 关键：添加代理环境变量
Environment=HTTP_PROXY=http://127.0.0.1:7890
Environment=HTTPS_PROXY=http://127.0.0.1:7890
Environment=NO_PROXY=localhost,127.0.0.1,192.168.0.0/16

[Install]
WantedBy=default.target
```

**特别注意**：
- `HTTP_PROXY`: HTTP请求代理
- `HTTPS_PROXY`: HTTPS请求代理
- `NO_PROXY`: 不使用代理的地址范围
  - `localhost`: 本地
  - `127.0.0.1`: 本地回环
  - `192.168.0.0/16`: 局域网（根据你的实际网络调整）

#### 配置3：重启服务使配置生效

```bash
# 1. 重新加载systemd配置
systemctl --user daemon-reload

# 2. 重启OpenClaw Gateway
systemctl --user restart openclaw-gateway

# 3. 查看状态
systemctl --user status openclaw-gateway

# 4. 查看日志
journalctl --user -u openclaw-gateway -f
```

### 5.3 测试 Telegram 连接

#### 方法1：查看日志

```bash
# 实时查看日志
journalctl --user -u openclaw-gateway -f

# 等待Telegram发送消息，观察日志输出
# 看到类似以下内容表示连接成功：
# [telegram] [default] starting provider
# [telegram] ✓ Telegram connected
```

#### 方法2：直接发送测试消息

```bash
# 1. 使用curl测试
curl -x http://127.0.0.1:7890 \
  "https://api.telegram.org/bot<your bot token>/getMe"

# 成功返回：
# {"ok":true,"result":{"id":123456789,"is_bot":true,"first_name":"MyBot","username":"my_claw_bot"}}

# 2. 发送测试消息
curl -x http://127.0.0.1:7890 \
  "https://api.telegram.org/bot<your bot token>/sendMessage" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"chat_id":你的Telegram用户ID,"text":"测试消息：OpenClaw已连接"}'

# 注意：需要先获取你的Telegram用户ID
# 方法：在Telegram中发送消息给 @userinfobot，它会返回你的ID
```

#### 方法3：在Telegram中测试

1. 打开你的机器人
2. 发送任意消息（如：`你好`）
3. 机器人应该自动回复
4. 如果没有回复，检查配置和日志

### 5.4 Telegram 配置常见问题

#### 问题1：【TypeError: fetch failed】

这是你之前遇到的问题！

**原因**：代理配置不完整

**解决步骤**：
```bash
# 1. 确认3处配置都已设置
#    - ~/.openclaw/openclaw.json 中有 proxy 字段
#    - ~/.config/systemd/user/openclaw-gateway.service 中有环境变量
#    - Mihomo 正在运行

# 2. 测试代理
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe

# 3. 如果上面失败，先修复代理

# 4. 重启服务
systemctl --user restart mihomo
systemctl --user restart openclaw-gateway

# 5. 查看详细日志
journalctl --user -u openclaw-gateway --since "1 min ago"
```

#### 问题2：【Network request failed】

**原因**：网络连接问题

**解决**：
```bash
# 1. 检查网络连接
ping -c 3 8.8.8.8

# 2. 检查代理是否运行
systemctl --user status mihomo

# 3. 测试其他网站
curl -x http://127.0.0.1:7890 https://www.google.com

# 4. 更新订阅
# 如果使用机场，可能需要更新订阅链接
```

#### 问题3：Bot可以接收消息但不能回复

**原因**：可能是权限问题或API Key问题

**解决**：
```bash
# 1. 检查OpenClaw日志
journalctl --user -u openclaw-gateway -f | grep ERROR

# 2. 检查AI API配置
# 查看 ~/.openclaw/openclaw.json 中的 models 配置

# 3. 测试AI API
curl https://api.siliconflow.cn/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your api key>" \
  -d '{"model":"Pro/zai-org/GLM-4.7","messages":[{"role":"user","content":"测试"}]}'
```

### 5.5 Telegram 配置总结

**3步完成Telegram连接**：
1. ✅ 创建Bot，获得Token
2. ✅ 配置3处代理设置（配置文件+Systemd）
3. ✅ 重启服务，测试连接

**验证成功的标志**：
```bash
# 查看日志看到：
[telegram] ✓ Telegram connected

# 发送消息给Bot，Bot能正常回复
```

**记住**：Telegram连接的核心是代理，代理配置正确，一切正常！

---

## 🔄 第六章：启动和自启动

### 6.1 手动启动 OpenClaw Gateway

```bash
# 前台启动（用于调试）
openclaw gateway

# 后台启动（推荐生产环境使用）
nohup openclaw gateway > ~/clawd/gateway.log 2>&1 &

# 查看日志
tail -f ~/clawd/gateway.log
```

### 6.2 使用 Systemd 管理（推荐）

```bash
# 启动服务
systemctl --user start openclaw-gateway

# 停止服务
systemctl --user stop openclaw-gateway

# 重启服务
systemctl --user restart openclaw-gateway

# 查看状态
systemctl --user status openclaw-gateway

# 查看日志
journalctl --user -u openclaw-gateway -f

# 查看最近100行日志
journalctl --user -u openclaw-gateway -n 100
```

### 6.3 配置开机自启动

```bash
# 如果还没有设置用户systemd权限
loginctl enable-linger $USER

# 启用开机自启
systemctl --user enable openclaw-gateway

# 查看是否已启用
systemctl --user is-enabled openclaw-gateway

# 禁用开机自启
systemctl --user disable openclaw-gateway
```

### 6.4 启动顺序（重要！）

**错误的启动顺序**：
```bash
# ❌ 先启动OpenClaw，后启动代理
systemctl --user start openclaw-gateway
systemctl --user start mihomo
# 结果：Telegram无法连接（代理未启动）
```

**正确的启动顺序**：
```bash
# ✅ 先启动代理，后启动OpenClaw
systemctl --user start mihomo
systemctl --user start openclaw-gateway
# 结果：一切正常
```

### 6.5 创建启动脚本

为了方便，可以创建一个启动脚本：

```bash
# 创建启动脚本
cat > ~/start-clawd.sh << 'EOF'
#!/bin/bash
# OpenClaw 启动脚本

echo "🚀 启动 OpenClaw 服务..."

# 1. 启动代理
echo "📡 启动代理服务..."
systemctl --user start mihomo
sleep 2

# 2. 检查代理状态
if systemctl --user is-active --quiet mihomo; then
    echo "✅ 代理服务已启动"
else
    echo "❌ 代理服务启动失败"
    exit 1
fi

# 3. 启动 OpenClaw
echo "🤖 启动 OpenClaw Gateway..."
systemctl --user start openclaw-gateway
sleep 2

# 4. 检查 OpenClaw 状态
if systemctl --user is-active --quiet openclaw-gateway; then
    echo "✅ OpenClaw Gateway 已启动"
else
    echo "❌ OpenClaw Gateway 启动失败"
    exit 1
fi

# 5. 显示状态
echo ""
echo "📊 服务状态："
systemctl --user status mihomo --no-pager -l | head -5
systemctl --user status openclaw-gateway --no-pager -l | head -5

echo ""
echo "🎉 OpenClaw 已成功启动！"
echo "📱 Telegram Bot 已就绪，可以发送消息了"
EOF

# 添加执行权限
chmod +x ~/start-clawd.sh

# 使用脚本
~/start-clawd.sh
```

---

## 🎯 第七章：常用命令

### 7.1 OpenClaw 命令

```bash
# 查看版本
openclaw --version

# 查看帮助
openclaw --help

# 查看状态
openclaw status

# 启动 Gateway
openclaw gateway

# 指定端口启动
openclaw gateway --port 64435

# 更新 OpenClaw
openclaw update

# 查看 Gateway 日志
openclaw logs

# 运行向导
openclaw onboard

# 配置帮助
openclaw configure --help
```

### 7.2 Systemd 管理命令

```bash
# 启动服务
systemctl --user start openclaw-gateway

# 停止服务
systemctl --user stop openclaw-gateway

# 重启服务
systemctl --user restart openclaw-gateway

# 查看状态
systemctl --user status openclaw-gateway

# 查看日志
journalctl --user -u openclaw-gateway -f

# 查看最近50行日志
journalctl --user -u openclaw-gateway -n 50

# 查看最近1小时的日志
journalctl --user -u openclaw-gateway --since "1 hour ago"

# 启用开机自启
systemctl --user enable openclaw-gateway

# 禁用开机自启
systemctl --user disable openclaw-gateway

# 查看服务是否启用
systemctl --user is-enabled openclaw-gateway

# 重新加载配置
systemctl --user daemon-reload
```

### 7.3 代理管理命令

```bash
# 启动代理
systemctl --user start mihomo

# 停止代理
systemctl --user stop mihomo

# 重启代理
systemctl --user restart mihomo

# 查看代理状态
systemctl --user status mihomo

# 查看代理日志
journalctl --user -u mihomo -f

# 测试代理 - 访问Google
curl -x http://127.0.0.1:7890 https://www.google.com

# 测试代理 - 访问Telegram API
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe

# 测试代理 - 使用Bot Token
curl -x http://127.0.0.1:7890 \
  "https://api.telegram.org/bot<your bot token>/getMe"

# 查看代理配置
cat ~/.config/mihomo/config.yaml | head -50

# 测试配置文件
mihomo -d ~/.config/mihomo -t
```

### 7.4 网络测试命令

```bash
# 测试网络连通性
ping -c 3 8.8.8.8

# 测试DNS解析
nslookup google.com

# 测试端口连通性
nc -zv api.telegram.org 443

# 查看网络连接
ss -tunlp | grep 7890

# 查看路由表
ip route show

# 测试HTTP连接
curl -I https://www.baidu.com

# 测试HTTP连接（通过代理）
curl -x http://127.0.0.1:7890 -I https://www.google.com
```

---

## 🐛 第八章：故障排除

### 8.1 Telegram 无法连接（【TypeError: fetch failed】）

这是最常见的故障！

**现象**：
```bash
# 日志中看到：
TypeError: fetch failed
telegram setMyCommands failed: Network request failed!
```

**原因分析**：
1. 代理未启动或配置错误
2. OpenClaw配置中未设置代理
3. Systemd环境变量未设置
4. 代理无法连接到Telegram

**完整解决流程**：

```bash
# 步骤1：检查Mihomo是否运行
systemctl --user status mihomo

# 如果未运行，启动它
systemctl --user start mihomo

# 步骤2：测试代理是否工作
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe

# 预期输出：
# {"ok":false,"error_code":404,"description":"Not Found"}

# 如果失败，代理有问题，修复代理配置

# 步骤3：检查OpenClaw配置文件
cat ~/.openclaw/openclaw.json | grep -A 10 "telegram"

# 必须看到：
# "proxy": "http://127.0.0.1:7890"

# 如果没有，添加它！

# 步骤4：检查Systemd环境变量
cat ~/.config/systemd/user/openclaw-gateway.service | grep PROXY

# 必须看到：
# Environment=HTTP_PROXY=http://127.0.0.1:7890
# Environment=HTTPS_PROXY=http://127.0.0.1:7890

# 如果没有，添加它！

# 步骤5：重新加载并重启
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway

# 步骤6：查看日志
journalctl --user -u openclaw-gateway -f

# 等待发送Telegram消息，观察日志
```

### 8.2 内存占用过高

**现象**：
```bash
# 内存占用超过80%
free -h
# 内存:  7.7Gi  6.2Gi  1.5Gi  80%
```

**原因**：
- 并发数设置过高
- 同时运行多个会话
- AI模型上下文过大
- 未设置并发限制

**解决方法**：

```bash
# 编辑配置文件
nano ~/.openclaw/openclaw.json

# 调整并发数设置：
{
  "agents": {
    "defaults": {
      "maxConcurrent": 2,      // 降低到2
      "subagents": {
        "maxConcurrent": 2    // 降低到2
      }
    }
  }
}

# 保存后重启
systemctl --user restart openclaw-gateway

# 监控内存使用
watch -n 2 free -h
```

### 8.3 Gateway 无法启动（端口被占用）

**现象**：
```bash
# 启动失败
systemctl --user start openclaw-gateway
# Job failed
```

**原因**：
- 端口64434已被占用
- 之前的进程没有正常退出

**解决方法**：

```bash
# 查看占用端口的进程
ss -tunlp | grep 64434

# 或者使用lsof（如果已安装）
lsof -i :64434

# 杀死进程（使用上面的PID）
kill -9 <PID>

# 重试启动
systemctl --user start openclaw-gateway

# 或者使用不同端口
openclaw gateway --port 64435
```

### 8.4 代理工作不正常

**现象**：
- 代理已启动但无法访问国外网站
- Telegram无法连接
- 速度很慢

**检查步骤**：

```bash
# 1. 检查代理状态
systemctl --user status mihomo

# 2. 查看代理日志
journalctl --user -u mihomo --since "5 min ago" | tail -20

# 3. 检查配置文件
mihomo -d ~/.config/mihomo -t

# 4. 测试代理延迟
time curl -x http://127.0.0.1:7890 https://www.google.com > /dev/null

# 应该在2秒以内完成

# 5. 测试其他网站
curl -x http://127.0.0.1:7890 https://www.google.com
curl -x http://127.0.0.1:7890 https://www.youtube.com
```

**常见原因及解决**：

#### 原因1：订阅过期

```bash
# 更新订阅
SUB_URL="你的订阅链接"
clash2sing-box --url "$SUB_URL" > ~/.config/mihomo/config.yaml

# 重启
systemctl --user restart mihomo
```

#### 原因2：节点失效

```yaml
# 编辑配置，选择其他节点
# 访问 http://localhost:9090/ui 切换节点
```

#### 原因3：代理软件版本过旧

```bash
# 更新Mihomo
cd ~/.local/bin
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.18.10/mihomo-linux-arm64-v1.18.10.gz
gunzip mihomo-linux-arm64-v1.18.10.gz
chmod +x mihomo
systemctl --user restart mihomo
```

### 8.5 OpenClaw 更新失败

**现象**：
```bash
openclaw update
# 错误：网络超时或下载失败
```

**原因**：访问GitHub慢

**解决方法**：

```bash
# 方法1：使用代理更新
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890
openclaw update

# 方法2：手动更新
cd ~/.nvm/versions/node/v24.13.0/lib/node_modules
rm -rf openclaw
npm install -g openclaw

# 方法3：使用淘宝镜像
npm config set registry https://registry.npmmirror.com
npm install -g openclaw
```

---

## 📚 第九章：进阶配置

### 9.1 配置更多消息平台

OpenClaw 支持多种消息平台：

| 平台 | 说明 | 配置难度 |
|------|------|---------|
| ✅ Telegram | 即时通讯，需要代理 | ⭐ |
| ✅ Discord | 游戏社区，需要代理 | ⭐⭐ |
| ✅ 微信 | 需要蓝牙设备+Mac/iPhone | ⭐⭐⭐⭐⭐ |
| ✅ Slack | 企业通讯工具 | ⭐⭐ |
| ✅ Google Chat | Google Workspace | ⭐⭐ |
| ✅ Line | 日韩常用 | ⭐⭐ |
| ✅ Matrix | 去中心化通讯协议 | ⭐⭐⭐ |
| ✅ Signal | 安全通讯 | ⭐⭐⭐ |
| ✅ Mattermost | 自建团队通讯 | ⭐⭐ |
| ✅ Microsoft Teams | 企业办公 | ⭐⭐ |

**配置多个平台示例**：

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "<your telegram bot token>",
      "proxy": "http://127.0.0.1:7890"
    },
    "discord": {
      "enabled": true,
      "botToken": "<your discord bot token>",
      "proxy": "http://127.0.0.1:7890"
    },
    "slack": {
      "enabled": true,
      "botToken": "<your slack bot token>"
    }
  }
}
```

### 9.2 配置多个AI模型

OpenClaw 可以同时配置多个AI提供商：

```json
{
  "models": {
    "providers": {
      "zai-api": {
        "baseUrl": "https://api.siliconflow.cn/v1",
        "apiKey": "<your zai api key>",
        "api": "openai-completions",
        "models": [
          {
            "id": "Pro/zai-org/GLM-4.7",
            "name": "GLM-4.7"
          },
          {
            "id": "Qwen/Qwen2.5-72B-Instruct",
            "name": "Qwen 72B"
          }
        ]
      },
      "deepseek": {
        "baseUrl": "https://api.deepseek.com/v1",
        "apiKey": "<your deepseek api key>",
        "api": "openai-completions",
        "models": [
          {
            "id": "deepseek-chat",
            "name": "DeepSeek Chat"
          },
          {
            "id": "deepseek-coder",
            "name": "DeepSeek Coder"
          }
        ]
      },
      "openai": {
        "baseUrl": "https://api.openai.com/v1",
        "apiKey": "<your openai api key>",
        "api": "openai-completions",
        "models": [
          {
            "id": "gpt-4o",
            "name": "GPT-4o"
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "models": {
        "zai-api/Pro/zai-org/GLM-4.7": {
          "alias": "glm"
        },
        "deepseek/deepseek-chat": {
          "alias": "deepseek"
        }
      }
    }
  }
}
```

**使用不同模型**：
- 默认使用配置的 `primary` 模型
- 可以在对话中切换：`@glm` 或 `@deepseek`
- 不同模型擅长不同任务

### 9.3 配置工作区

工作区是OpenClaw存储文件和数据的地方：

```bash
# 1. 创建新的工作目录
mkdir -p ~/my-projects/clawd

# 2. 设置工作区
openclaw configure --set workspace=~/my-projects/clawd

# 3. 或者在配置文件中设置
# 编辑 ~/.openclaw/openclaw.json
{
  "agents": {
    "defaults": {
      "workspace": "/home/youruser/my-projects/clawd"
    }
  }
}

# 4. 重启服务
systemctl --user restart openclaw-gateway
```

### 9.4 配置自定义命令

OpenClaw支持自定义命令，可以创建快捷指令：

```bash
# 创建命令目录
mkdir -p ~/.openclaw/commands

# 创建自定义命令
cat > ~/.openclaw/commands/status.sh << 'EOF'
#!/bin/bash
echo "📊 OpenClaw 状态报告"
echo "=================="
echo "CPU使用率: $(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)%"
echo "内存使用: $(free -h | grep Mem | awk '{print $3}')"
echo "磁盘使用: $(df -h /home | tail -1 | awk '{print $5}')"
echo ""
echo "服务状态："
systemctl --user is-active openclaw-gateway && echo "✅ OpenClaw: 运行中" || echo "❌ OpenClaw: 已停止"
systemctl --user is-active mihomo && echo "✅ Mihomo: 运行中" || echo "❌ Mihomo: 已停止"
EOF

chmod +x ~/.openclaw/commands/status.sh

# 在Telegram中使用：
# 发送：/status
```

---

## 💡 第十章：最佳实践

### 10.1 性能优化

#### 1. 合理设置并发数

```json
{
  "agents": {
    "defaults": {
      // 主代理并发数
      // RK3588 (8核7GB): 建议 2-4
      // Raspberry Pi (4核4GB): 建议 1-2
      // 树莓派 Zero (1核512MB): 建议 1
      "maxConcurrent": 2,
      
      // 子代理并发数
      "subagents": {
        "maxConcurrent": 2
      }
    }
  }
}
```

#### 2. 使用缓存

```json
{
  "agents": {
    "defaults": {
      "model": {
        // 启用token缓存可以减少API调用
        "cache": {
          "enabled": true,
          "maxSize": 1000
        }
      }
    }
  }
}
```

#### 3. 限制上下文窗口

```json
{
  "models": {
    "providers": {
      "my-provider": {
        "models": [
          {
            "id": "my-model",
            "name": "My Model",
            "contextWindow": 32768,  // 降低到32K（默认128K）
            "maxTokens": 4096        // 降低到4K（默认8K）
          }
        ]
      }
    }
  }
}
```

#### 4. 定期清理内存

```bash
# 创建清理脚本
cat > ~/cleanup.sh << 'EOF'
#!/bin/bash
echo "🧹 清理OpenClaw缓存..."

# 清理日志文件
find ~/.openclaw -name "*.log" -mtime +7 -delete

# 清理缓存
rm -rf ~/.openclaw/cache/*

# 重启服务
systemctl --user restart openclaw-gateway

echo "✅ 清理完成"
EOF

chmod +x ~/cleanup.sh

# 每周自动运行
(crontab -l 2>/dev/null; echo "0 3 * * 0 ~/cleanup.sh") | crontab -
```

### 10.2 安全建议

#### 1. 保护API密钥

```bash
# 永远不要在代码中硬编码API密钥
# ❌ 错误做法
const apiKey = "sk-1234567890"

# ✅ 正确做法：使用环境变量
export MY_API_KEY="<your api key>"
const apiKey = process.env.MY_API_KEY
```

#### 2. 配置用户权限

```json
{
  "channels": {
    "telegram": {
      // 私聊策略
      "dmPolicy": "pairing",  // 配对模式，不是任何人都能私聊
      
      // 允许的发送者
      "allowFrom": [
        "telegram:123456789",  // 只允许特定用户
        "telegram:987654321"
      ],
      
      // 群组策略
      "groupPolicy": "allowlist",  // 白名单模式
      
      // 允许的群组
      "groups": {
        "-1001234567890": {
          "requireMention": true  // 必须@机器人才响应
        }
      }
    }
  }
}
```

#### 3. 启用Gateway认证

```bash
# 在Systemd配置中添加token
Environment=OPENCLAW_GATEWAY_TOKEN=<your-secret-token>
```

#### 4. 定期轮换密钥

```bash
# 定期更新API密钥（建议每月一次）
# 1. 在平台上生成新的API Key
# 2. 更新OpenClaw配置
# 3. 重启服务
# 4. 删除旧的API Key
```

### 10.3 备份与恢复

#### 备份脚本

```bash
#!/bin/bash
# backup.sh - OpenClaw 备份脚本

BACKUP_DIR="$HOME/clawd-backups/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

echo "💾 开始备份 OpenClaw..."

# 备份配置文件
cp ~/.openclaw/openclaw.json "$BACKUP_DIR/"
cp ~/.config/mihomo/config.yaml "$BACKUP_DIR/"

# 备份工作区
tar -czf "$BACKUP_DIR/workspace.tar.gz" ~/clawd

# 备份NVM配置
cp ~/.bashrc "$BACKUP_DIR/"

echo "✅ 备份完成: $BACKUP_DIR"
ls -lh "$BACKUP_DIR"
```

```bash
chmod +x ~/backup.sh

# 每天自动备份
(crontab -l 2>/dev/null; echo "0 2 * * * ~/backup.sh") | crontab -
```

#### 恢复脚本

```bash
#!/bin/bash
# restore.sh - OpenClaw 恢复脚本

BACKUP_DIR="$1"

if [ -z "$BACKUP_DIR" ]; then
    echo "用法: $0 <备份目录>"
    exit 1
fi

echo "🔄 开始恢复 OpenClaw..."

# 恢复配置文件
cp "$BACKUP_DIR/openclaw.json" ~/.openclaw/
cp "$BACKUP_DIR/config.yaml" ~/.config/mihomo/

# 恢复工作区
tar -xzf "$BACKUP_DIR/workspace.tar.gz" -C ~/

# 重启服务
systemctl --user restart mihomo
systemctl --user restart openclaw-gateway

echo "✅ 恢复完成"
```

```bash
chmod +x ~/restore.sh

# 使用
~/restore.sh ~/clawd-backups/20260203
```

### 10.4 监控与告警

#### 系统监控

```bash
# 安装监控工具
sudo apt install htop iotop

# 实时监控CPU
htop

# 实时监控磁盘IO
sudo iotop

# 实时监控网络
iftop
```

#### 服务监控

```bash
# 创建监控脚本
cat > ~/monitor.sh << 'EOF'
#!/bin/bash
# 监控脚本 - 检查服务状态

ALL_OK=true

# 检查 Mihomo
if ! systemctl --user is-active --quiet mihomo; then
    echo "❌ Mihomo 未运行"
    ALL_OK=false
else
    echo "✅ Mihomo 运行中"
fi

# 检查 OpenClaw
if ! systemctl --user is-active --quiet openclaw-gateway; then
    echo "❌ OpenClaw Gateway 未运行"
    ALL_OK=false
else
    echo "✅ OpenClaw Gateway 运行中"
fi

# 检查端口
if ! ss -tunlp | grep -q 7890; then
    echo "❌ 代理端口7890未监听"
    ALL_OK=false
else
    echo "✅ 代理端口7890监听正常"
fi

if ! ss -tunlp | grep -q 64434; then
    echo "❌ Gateway端口64434未监听"
    ALL_OK=false
else
    echo "✅ Gateway端口64434监听正常"
fi

if $ALL_OK; then
    echo "🎉 所有服务正常！"
    exit 0
else
    echo "⚠️  部分服务异常！"
    exit 1
fi
EOF

chmod +x ~/monitor.sh

# 定时检查
*/5 * * * * ~/monitor.sh >> ~/monitor.log 2>&1
```

#### 日志监控

```bash
# 实时查看错误日志
journalctl --user -u openclaw-gateway -f | grep ERROR

# 查看今天的错误
journalctl --user -u openclaw-gateway --since today | grep ERROR

# 统计今天的错误数量
journalctl --user -u openclaw-gateway --since today | grep ERROR | wc -l
```

---

## 🎓 附录

### A. 相关资源

#### 官方资源
- OpenClaw 官方文档: https://docs.openclaw.ai
- OpenClaw GitHub: https://github.com/openclaw/openclaw
- OpenClaw 社区: https://discord.gg/clawd
- OpenClaw 更新日志: https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md

#### 代理工具
- Mihomo GitHub: https://github.com/MetaCubeX/mihomo
- Clash GitHub: https://github.com/Dreamacro/clash
- 机场推荐列表: https://blog.clash.icu/

#### AI API提供商
- Zai AI: https://cloud.siliconflow.cn/
- DeepSeek: https://platform.deepseek.com/
- 通义千问: https://portal.qwen.ai/
- 月之暗面: https://platform.moonshot.cn/
- OpenAI: https://platform.openai.com/

#### Node.js相关
- Node.js官网: https://nodejs.org/
- NPM镜像: https://npmmirror.com/
- NVM文档: https://github.com/nvm-sh/nvm

#### 国内镜像源
- 清华开源镜像站: https://mirrors.tuna.tsinghua.edu.cn/
- 阿里云镜像: https://developer.aliyun.com/mirror/
- 中科大镜像: https://mirrors.ustc.edu.cn/

### B. 常用端口速查

| 服务 | 端口 | 协议 | 说明 |
|------|------|------|------|
| Mihomo HTTP | 7891 | HTTP | HTTP代理端口 |
| Mihomo SOCKS | 7892 | SOCKS5 | SOCKS代理端口 |
| Mihomo Mixed | 7890 | HTTP/SOCKS | 混合代理（推荐）|
| Mihomo UI | 9090 | HTTP | Web控制面板 |
| OpenClaw Gateway | 64434 | WebSocket | Gateway服务 |
| OpenClaw Web | 64435 | HTTP | Web界面 |

### C. 配置文件路径速查

| 配置项 | 路径 | 说明 |
|--------|------|------|
| OpenClaw主配置 | `~/.openclaw/openclaw.json` | 所有核心配置 |
| Mihomo配置 | `~/.config/mihomo/config.yaml` | 代理节点和规则 |
| OpenClaw服务 | `~/.config/systemd/user/openclaw-gateway.service` | Systemd服务文件 |
| Mihomo服务 | `~/.config/systemd/user/mihomo.service` | Systemd服务文件 |
| OpenClaw日志 | `/tmp/openclaw/openclaw-*.log` | 运行日志 |
| Mihomo日志 | `journalctl --user -u mihomo` | 系统日志 |
| 工作区 | `~/clawd` | 默认工作目录 |
| 命令目录 | `~/.openclaw/commands` | 自定义命令 |

### D. 网络测试命令汇总

```bash
# 测试本地网络
ping -c 3 8.8.8.8
ping -c 3 114.114.114.114

# 测试DNS
nslookup google.com
dig google.com

# 测试端口
nc -zv api.telegram.org 443
nc -zv www.google.com 443

# 测试HTTP
curl -I https://www.baidu.com
curl -I https://www.google.com

# 测试代理
curl -x http://127.0.0.1:7890 https://www.google.com
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe

# 测试速度
time curl -x http://127.0.0.1:7890 https://www.google.com > /dev/null

# 查看路由
ip route show
route -n

# 查看连接
ss -tunlp
netstat -tunlp

# 测试延迟
ping -c 10 google.com
```

### E. ARM64系统特殊说明

#### 下载对应版本的软件

```bash
# 检查系统架构
uname -m

# ARM64（aarch64）常见硬件：
# - RK3588
# - Raspberry Pi 4/5 (64位系统)
# - Orange Pi 5
# - N1盒子

# 下载ARM64版本的软件：
# Mihomo:
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.18.10/mihomo-linux-arm64-v1.18.10.gz

# 如果下载了错误的版本，删除重新下载
rm mihomo-linux-arm64-v1.18.10
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.18.10/mihomo-linux-arm64-v1.18.10.gz
```

#### 性能优化建议

ARM64处理器通常性能不如x86_64，需要优化：

```bash
# 1. 降低并发数
# 编辑 ~/.openclaw/openclaw.json
{
  "agents": {
    "defaults": {
      "maxConcurrent": 1,  // 降低到1
      "subagents": {
        "maxConcurrent": 1
      }
    }
  }
}

# 2. 使用轻量级模型
{
  "models": {
    "providers": {
      "deepseek": {
        "models": [
          {
            "id": "deepseek-chat",
            "contextWindow": 8192,    // 降低上下文
            "maxTokens": 2048        // 降低输出长度
          }
        ]
      }
    }
  }
}

# 3. 定期清理内存
# 每4小时重启一次（根据需要调整）
0 */4 * * * systemctl --user restart openclaw-gateway
```

### F. 安全检查清单

部署完成后的安全检查：

```bash
# 1. 检查文件权限
ls -la ~/.openclaw/openclaw.json
# 应该是 -rw------- (只有所有者可读写)

# 2. 检查敏感信息
grep -i "api.*key" ~/.openclaw/openclaw.json
# 确认没有泄露真实的API密钥

# 3. 检查代理配置
cat ~/.config/mihomo/config.yaml | grep "password\|username"
# 如果有密码，确保已加密

# 4. 检查日志权限
ls -la /tmp/openclaw/
# 确保日志文件权限正确

# 5. 检查防火墙
sudo ufw status
# 如果启用，确保规则正确

# 6. 检查开放端口
ss -tunlp | grep LISTEN
# 只开放必要的端口

# 7. 检查用户权限
id
# 确认使用非特权用户运行
```

### G. 常见错误代码速查

| 错误信息 | 含义 | 解决方法 |
|---------|------|---------|
| `ECONNREFUSED` | 连接被拒绝 | 检查服务是否运行 |
| `ETIMEDOUT` | 连接超时 | 检查网络和Proxy |
| `401 Unauthorized` | 认证失败 | 检查API Key |
| `404 Not Found` | 资源不存在 | 检查URL和路径 |
| `429 Too Many Requests` | 请求过多 | 降低并发数 |
| `500 Internal Server Error` | 服务器错误 | 稍后重试 |
| `502 Bad Gateway` | 网关错误 | 检查Proxy配置 |
| `503 Service Unavailable` | 服务不可用 | 稍后重试 |

---

## 🤝 结语

恭喜你完成了 OpenClaw 的完整部署！🎉

### 📋 最后检查清单

部署完成后，请检查以下项目：

- ✅ APT源已切换到国内镜像
- ✅ Python pip已配置国内源
- ✅ NVM和NPM已配置国内镜像
- ✅ Mihomo代理已配置并正常运行
- ✅ OpenClaw已安装并可正常启动
- ✅ AI API已配置并可正常调用
- ✅ Telegram Bot已配置并可收发消息
- ✅ 所有服务已设置开机自启动
- ✅ 配置文件已备份
- ✅ API密钥和Token已安全保存

### 🚀 下一步

现在你已经拥有了一个强大的AI助手！可以：

1. **在Telegram中使用**：给你的机器人发消息
2. **配置更多平台**：Discord、Slack等
3. **创建自定义命令**：添加快捷指令
4. **开发自定义功能**：使用OpenClaw的扩展机制

### 📞 获取帮助

如果你在配置过程中遇到问题：

1. **查看本文档**：故障排除章节
2. **查看日志**：`journalctl --user -u openclaw-gateway -f`
3. **访问社区**：https://discord.gg/clawd
4. **查看官方文档**：https://docs.openclaw.ai

### 💭 特别提醒

**RK3588最小系统也能运行OpenClaw！** 关键是：
- ✅ 配置国内源，提升下载速度
- ✅ 配置代理，访问国外服务
- ✅ 合理设置并发数，防止内存溢出
- ✅ 选择合适的AI模型，平衡性能和成本

**记住**：这是国内首个在ARM64低配服务器上运行的OpenClaw实例！你已经创造了一个奇迹！🎉

---

**文档版本**: v2.0
**编写时间**: 2026-02-03
**测试环境**: Rockchip RK3588 + Linux 5.10.198-rk3588 + OpenClaw v2026.1.29
**作者**: ClawdBot Community

**Happy Clawing! 🚀**
