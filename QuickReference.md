# OpenClaw 快速参考卡

## 🔥 核心配置速查

### 1️⃣ APT国内源
```bash
# 备份
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup

# 清华源（Ubuntu 22.04）
sudo tee /etc/apt/sources.list > /dev/null <<EOF
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
EOF

sudo apt update
```

### 2️⃣ Python PIP国内源
```bash
mkdir -p ~/.pip
cat > ~/.pip/pip.conf <<EOF
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
trusted-host = pypi.tuna.tsinghua.edu.cn
EOF
```

### 3️⃣ NVM国内镜像
```bash
# 安装NVM
curl -o- https://gitee.com/mirrors/nvm/raw/master/install.sh | bash
source ~/.bashrc

# 配置镜像
echo 'export NVM_NODEJS_ORG_MIRROR=https://npmmirror.com/mirrors/node/' >> ~/.bashrc
source ~/.bashrc
```

### 4️⃣ NPM国内源
```bash
npm config set registry https://registry.npmmirror.com
```

### 5️⃣ 代理配置必须项
```yaml
# ~/.config/mihomo/config.yaml
mixed-port: 7890  # 混合代理端口
```

```json
// ~/.openclaw/openclaw.json
{
  "channels": {
    "telegram": {
      "botToken": "<your bot token>",
      "proxy": "http://127.0.0.1:7890"  // 必须配置！
    }
  },
  "models": {
    "providers": {
      "zai-api": {
        "baseUrl": "https://api.siliconflow.cn/v1",
        "apiKey": "<your api key>"
      }
    }
  }
}
```

```ini
# ~/.config/systemd/user/openclaw-gateway.service
Environment=HTTP_PROXY=http://127.0.0.1:7890
Environment=HTTPS_PROXY=http://127.0.0.1:7890
Environment=NO_PROXY=localhost,127.0.0.1,192.168.0.0/16
```

## 🎯 常用端口
| 服务 | 端口 | 说明 |
|------|------|------|
| Mihomo Mixed | 7890 | 混合代理（推荐） |
| Mihomo HTTP | 7891 | HTTP代理 |
| Mihomo SOCKS | 7892 | SOCKS5代理 |
| Mihomo UI | 9090 | Web控制面板 |
| OpenClaw Gateway | 64434 | Gateway服务 |

## 🚀 启动命令
```bash
# 启动代理
systemctl --user start mihomo

# 启动 OpenClaw
systemctl --user start openclaw-gateway

# 查看状态
systemctl --user status openclaw-gateway

# 查看日志
journalctl --user -u openclaw-gateway -f
```

## 🐛 快速诊断
```bash
# 测试代理
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe

# 测试 Telegram Bot
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe?bot_token=<your bot token>

# 查看端口占用
ss -tunlp | grep 64434

# 检查服务状态
systemctl --user status mihomo
systemctl --user status openclaw-gateway
```

## 📋 配置文件路径
| 配置项 | 路径 |
|--------|------|
| OpenClaw 主配置 | `~/.openclaw/openclaw.json` |
| Mihomo 配置 | `~/.config/mihomo/config.yaml` |
| OpenClaw 服务 | `~/.config/systemd/user/openclaw-gateway.service` |
| Mihomo 服务 | `~/.config/systemd/user/mihomo.service` |
| 工作区 | `~/clawd` |

## 🎯 常见问题速解

### ❌ Telegram 无法连接
```bash
# 1. 检查代理
systemctl --user status mihomo

# 2. 测试代理
curl -x http://127.0.0.1:7890 https://api.telegram.org/getMe

# 3. 检查配置中是否设置 proxy 字段
cat ~/.openclaw/openclaw.json | grep proxy

# 4. 检查 systemd 是否设置环境变量
cat ~/.config/systemd/user/openclaw-gateway.service | grep PROXY

# 5. 重启服务
systemctl --user restart mihomo
systemctl --user restart openclaw-gateway
```

### ❌ 内存占用高
编辑 `~/.openclaw/openclaw.json`:
```json
{
  "agents": {
    "defaults": {
      "maxConcurrent": 2,
      "subagents": {
        "maxConcurrent": 2
      }
    }
  }
}
```

### ❌ 下载速度慢
```bash
# 检查所有源是否已配置国内
# 1. APT
cat /etc/apt/sources.list | grep tsinghua

# 2. PIP
cat ~/.pip/pip.conf

# 3. NPM
npm config get registry

# 4. NVM
echo $NVM_NODEJS_ORG_MIRROR
```

## 🔧 安装速查
```bash
# NVM（使用Gitee镜像）
curl -o- https://gitee.com/mirrors/nvm/raw/master/install.sh | bash
source ~/.bashrc
nvm install --lts

# NPM（使用淘宝镜像）
npm config set registry https://registry.npmmirror.com

# OpenClaw
npm install -g openclaw

# Mihomo (ARM64)
cd ~/.local/bin
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.18.10/mihomo-linux-arm64-v1.18.10.gz
gunzip mihomo-linux-arm64-v1.18.10.gz
chmod +x mihomo
cd ~
```

## 🔑 API替代配置
```json
{
  "models": {
    "providers": {
      "zai-api": {
        "baseUrl": "https://api.siliconflow.cn/v1",
        "apiKey": "<your api key>",
        "models": [
          {"id": "Pro/zai-org/GLM-4.7", "name": "GLM-4.7"}
        ]
      },
      "deepseek": {
        "baseUrl": "https://api.deepseek.com/v1",
        "apiKey": "<your api key>",
        "models": [
          {"id": "deepseek-chat", "name": "DeepSeek"}
        ]
      },
      "qwen": {
        "baseUrl": "https://portal.qwen.ai/v1",
        "apiKey": "<your api key>",
        "models": [
          {"id": "qwen-plus", "name": "Qwen Plus"}
        ]
      }
    }
  }
}
```

## 📞 获取帮助
- 完整教程：`RK3588-OpenClafw-Guide.md`
- 官方文档：https://docs.openclaw.ai
- 社区：https://discord.gg/clawd
