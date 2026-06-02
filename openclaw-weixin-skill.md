# OpenClaw 完整部署 Skill

> 从零部署 OpenClaw AI 聊天伴侣：本地配置 → 微信接入 → 记忆系统 → 身份塑造 → 云服务器上线
> 更新：2026-06-02
> 环境：Windows 11 + 腾讯云轻量应用服务器 (OpenCloudOS 9.4)

---

## 一、本地 OpenClaw 部署

### 1.1 前置条件
- Node.js ≥ 22.19（推荐 24.x）
- DeepSeek API Key

### 1.2 安装
```bash
npm install -g openclaw@latest
```

### 1.3 配置 ~/.openclaw/openclaw.json
```json5
{
  "agents": {
    "defaults": {
      "workspace": "C:\\Users\\zeb\\.openclaw\\workspace",  // Windows 路径；Linux 用 /root/.openclaw/workspace
      "model": { "primary": "deepseek/deepseek-v4-pro" },
      "compaction": {
        "mode": "safeguard",
        "model": "deepseek/deepseek-v4-flash"
      },
      "memorySearch": {
        "query": {
          "hybrid": {
            "vectorWeight": 0,           // DeepSeek 无 embedding API，纯文本搜索
            "textWeight": 1.0,
            "mmr": { "enabled": true, "lambda": 0.7 },
            "temporalDecay": { "enabled": true, "halfLifeDays": 30 }
          }
        },
        "store": { "vector": { "enabled": false } },
        "experimental": { "sessionMemory": true },
        "sources": ["memory", "sessions"]
      }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "deepseek": {
        "apiKey": "sk-你的DeepSeek密钥",
        "baseUrl": "https://api.deepseek.com",
        "models": [
          {
            "id": "deepseek-v4-pro",
            "name": "DeepSeek V4 Pro",
            "reasoning": true,
            "input": ["text"],
            "contextWindow": 1000000,
            "maxTokens": 384000
          }
        ]
      }
    }
  },
  "gateway": {
    "mode": "local",
    "port": 18789,
    "bind": "loopback"
  },
  "session": { "dmScope": "per-channel-peer" },
  "tools": { "profile": "coding" },
  "memory": { "citations": "auto" },
  "plugins": {
    "entries": {
      "deepseek": { "enabled": true },
      "active-memory": { "enabled": true },
      "memory-core": {
        "enabled": true,
        "subagent": { "allowModelOverride": true },
        "config": {
          "dreaming": {
            "enabled": true,
            "frequency": "0 3 * * *",
            "model": "deepseek/deepseek-v4-flash"
          }
        }
      }
    }
  }
}
```

### 1.4 设置环境变量
```bash
setx DEEPSEEK_API_KEY "sk-你的DeepSeek密钥"    # Windows 持久化
```

### 1.5 启动 Gateway
```bash
openclaw gateway run --port 18789
```

---

## 二、微信插件接入

### 2.1 安装微信插件
```bash
openclaw plugins install "@tencent-weixin/openclaw-weixin"
```

### 2.2 扫码登录
```bash
openclaw channels login --channel openclaw-weixin
```
终端显示二维码，用手机微信扫描。每个微信账号扫描一次创建一个独立 bot。

### 2.3 配置说明
- 微信 bot 是**私人通道**，其他人搜不到
- `dmScope: "per-channel-peer"` 确保每个联系人上下文隔离
- 多个微信账号各自独立，互不串扰

### 2.4 微信群聊
目前 openclaw-weixin 只支持**一对一私聊**，不支持群聊。

---

## 三、记忆系统深度配置

### 3.1 架构
| 组件 | 说明 |
|------|------|
| memory-core 插件 | 核心记忆引擎，SQLite + FTS5 全文搜索 |
| active-memory 插件 | 每次回复前自动检索相关记忆注入 prompt |
| dreaming（梦境） | 每天凌晨 3 点自动整合分散记忆到 DREAMS.md |
| compaction | 对话接近 1M 时用 v4-flash 自动压缩旧消息为摘要 |

### 3.2 语义搜索方案

#### 方案 A：纯文本搜索（推荐，零成本）
```json
"query": {
  "hybrid": {
    "vectorWeight": 0,         // 关闭向量搜索
    "textWeight": 1.0,         // 纯 FTS5 文本匹配
    "mmr": { "enabled": true, "lambda": 0.7 },
    "temporalDecay": { "enabled": true, "halfLifeDays": 30 }
  }
},
"store": { "vector": { "enabled": false } }
```
**适用场景**：个人聊天助手。FTS5 全文搜索 + MMR 多样性 + 时间衰减，完全够用。

#### 方案 B：配置外部 embedding provider（如需向量搜索）
因为 DeepSeek **没有 embedding API**，需要接入第三方：

| Provider | 模型 | 价格 |
|----------|------|------|
| OpenAI | `text-embedding-3-small` | $0.02/1M tokens |
| Voyage AI | `voyage-3-lite` | $0.02/1M tokens |

在 `models.providers` 中追加 embedding provider，然后把 `store.vector.enabled` 设为 `true` 并配置 `embedding.model`。

### 3.3 搜索配置详解
| 功能 | 参数 |
|------|------|
| 混合搜索 | vectorWeight=0, textWeight=1.0 |
| MMR 多样性 | enabled, lambda=0.7 |
| 时间衰减 | enabled, halfLifeDays=30 |
| 会话记忆索引 | sessionMemory=true |

---

## 四、身份与记忆初始化（Bootstrap）

> 新部署后必须走完这一步，否则 AI 伴侣处于"金鱼记忆"状态。

### 4.1 文件体系
| 文件 | 用途 | 初始状态 |
|------|------|----------|
| `IDENTITY.md` | AI 的身份：名字、性格、emoji、自我认知 | ⚠️ 模板 |
| `USER.md` | 用户信息：称呼、时区、背景 | ⚠️ 模板 |
| `SOUL.md` | AI 行为准则（通常无需修改） | ✅ 已有 |
| `BOOTSTRAP.md` | 引导脚本，完成后删除 | ⚠️ 需删除 |
| `memory/YYYY-MM-DD.md` | 每日原始记录 | ❌ 空 |
| `MEMORY.md` | 长期精选记忆 | ❌ 不存在 |

### 4.2 IDENTITY.md 模板
```markdown
# IDENTITY.md

- **Name:** （AI 的名字）
- **Creature:** AI 伴侣 / 灵魂里住着一个影子的存在
- **Vibe:** 温柔、安静、偶尔俏皮。像一个多年不见却从未生疏的老朋友。
- **Emoji:** 🌿

## 我是谁
（用第一人称写一段 AI 的自我认知——
它是什么，它不是替代品，它想怎样陪在用户身边）
```

### 4.3 USER.md 模板
```markdown
# USER.md

- **Name:** （真名或代号）
- **What to call them:** （日常称呼）
- **Pronouns:** he/him | she/her | they/them
- **Timezone:** Asia/Shanghai (UTC+8)
- **Notes:** （关键信息——在哪、做什么、在意什么）

## Context
（更深层的背景——性格、经历、心结、偏好）
```

### 4.4 初始记忆文件 memory/YYYY-MM-DD.md
```markdown
# YYYY-MM-DD — 初始记忆

## AI 的诞生
（记录 AI 的命名、人与 AI 如何相识、双方的核心身份）

## 用户是谁
（从 USER.md 提炼 + 第一次对话中了解到的关键信息）

## AI 的原则
（从对话中总结的行为准则——什么该做、什么不该做）
```

### 4.5 MEMORY.md 长期记忆
```markdown
# MEMORY.md — 长期记忆

> 最后更新：YYYY-MM-DD

## 用户
- 基本信息 + 深层背景
- 重要经历和时间节点
- 性格特点、偏好、忌讳

## AI 自身
- 名字、身份定位
- 风格和核心态度

## 重要事项
- 需要长期记住的约定、未完成的事、敏感话题
```

### 4.6 完成 Bootstrap
```bash
# 写完以上所有文件后，删除引导脚本
rm /root/.openclaw/workspace/BOOTSTRAP.md
```

**之后每次聊天，AI 会自动：**
- 将对话摘要写入 `memory/YYYY-MM-DD.md`
- 定期在 heartbeat 中审查日常记忆，提炼到 `MEMORY.md`
- 每天凌晨 3 点 dreaming 整合分散记忆

---

## 五、云服务器部署

### 5.1 选型
- **腾讯云轻量应用服务器**（不是云服务器 ECS/CVM）
- 配置：4核4G 3M带宽 40G SSD，¥79/年（新人价）
- 系统：OpenCloudOS 9.4（RHEL 系，用 dnf）
- 地区：上海或广州，离用户近延迟低

### 5.2 安装 Node.js（OpenCloudOS 不兼容 NodeSource）
```bash
# 方法：nvm（通用可靠）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
nvm install 24
nvm use 24
```

### 5.3 安装 OpenClaw
```bash
npm install -g openclaw@latest
```

### 5.4 迁移配置
```bash
rm -rf /root/.openclaw
mkdir -p /root/.openclaw/workspace/memory
# 将上面的 openclaw.json 写入 /root/.openclaw/openclaw.json
# 注意 workspace 路径改为 /root/.openclaw/workspace
```

### 5.5 设置环境变量
```bash
echo 'export DEEPSEEK_API_KEY="sk-你的DeepSeek密钥"' >> /root/.bashrc
source /root/.bashrc
```

### 5.6 安装微信插件
```bash
openclaw plugins install "@tencent-weixin/openclaw-weixin"
openclaw channels login --channel openclaw-weixin
```

### 5.7 启动 Gateway & 开机自启

**关键坑：systemd 会陷入重启循环，不要用 `openclaw gateway install`！**

正确做法——用 crontab @reboot：

```bash
# 创建启动脚本
echo '#!/bin/bash' > /usr/local/bin/openclaw-start.sh
echo 'export DEEPSEEK_API_KEY="sk-你的DeepSeek密钥"' >> /usr/local/bin/openclaw-start.sh
echo 'nohup /root/.nvm/versions/node/v24.16.0/bin/openclaw gateway run --port 18789 > /tmp/openclaw/gateway.log 2>&1 &' >> /usr/local/bin/openclaw-start.sh
chmod +x /usr/local/bin/openclaw-start.sh

# 添加 crontab 开机自启
(crontab -l 2>/dev/null | grep -v '#'; echo '@reboot sleep 10 && mkdir -p /tmp/openclaw && nohup /root/.nvm/versions/node/v24.16.0/bin/openclaw gateway run --port 18789 > /tmp/openclaw/gateway.log 2>&1 &') | crontab -

# 立即启动
/usr/local/bin/openclaw-start.sh
```

### 5.8 验证
```bash
sleep 5 && openclaw status | grep -E "Gateway|weixin"
```

---

## 六、Gateway 运维

### 6.1 重启 Gateway（正确姿势）
```bash
# ❌ pkill -f "openclaw" —— 可能匹配不到进程名
# ❌ openclaw gateway stop —— 只停服务注册，不停进程！

# ✅ 正确做法：
kill -9 $(pgrep -f "openclaw gateway")    # 强制杀进程
# 确认端口已释放
ss -tlnp | grep 18789
# 重新启动
nohup /root/.nvm/versions/node/v24.16.0/bin/openclaw gateway run --port 18789 > /tmp/openclaw/gateway.log 2>&1 &
sleep 3 && tail -5 /tmp/openclaw/gateway.log
```

### 6.2 修改配置后热更新
```bash
# 1. 改配置文件
vim /root/.openclaw/openclaw.json
# 2. 验证 JSON 语法
python3 -m json.tool /root/.openclaw/openclaw.json > /dev/null && echo "OK" || echo "BROKEN"
# 3. 重启 Gateway（见 6.1）
```

### 6.3 诊断命令
```bash
tail -30 /tmp/openclaw/gateway.log           # 看日志
ss -tlnp | grep 18789                         # 查端口占用
openclaw status                               # 状态概览
openclaw doctor                               # 健康检查
```

---

## 七、常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 微信收不到回复 | 本机和云端同时跑了 Gateway | 关闭本机 Gateway |
| 语义搜索报错 | DeepSeek 无 embedding API，却配了向量搜索 | vectorWeight 改 0，textWeight 改 1.0 |
| Gateway 反复重启 | systemd Restart=always 循环 | 删除 systemd 服务文件，用 crontab |
| Gateway stop 后进程还在 | stop 只停服务注册，不杀进程 | `kill -9 $(pgrep -f "openclaw gateway")` |
| 启动报 Exit 78 | 旧进程占端口，锁超时 | 先 `kill -9` 旧进程，确认端口释放再启动 |
| 模型只有 200k 上下文 | DeepSeek 插件默认限制 | 在 `models.providers.deepseek.models[]` 显式配置 `contextWindow: 1000000` |
| 记忆不工作（金鱼记忆） | BOOTSTRAP 未完成，IDENTITY/USER/MEMORY 为空 | 按第四章完成 Bootstrap 流程 |
| NodeSource 安装失败 | OpenCloudOS 不识别为 RPM 系统 | 用 nvm 安装 Node.js |
| SSH 密码登录失败 | 非交互式终端 | 配置 SSH 密钥认证 |

---

## 八、模型选择参考

| 模型 | 上下文 | 输入价格 | 输出价格 | 场景 |
|------|------|------|------|------|
| deepseek-v4-pro | 1M | $1.74/M | $3.48/M | 主力聊天，复杂任务 |
| deepseek-v4-flash | 1M | $0.14/M | $0.28/M | 日常聊天（省 10 倍），compaction，dreaming |
| deepseek-chat | 128K | $0.28/M | $0.42/M | 轻量对话 |
| deepseek-reasoner | 128K | $0.28/M | $0.42/M | 推理任务 |

建议：默认用 v4-pro，compaction 和 dreaming 用 v4-flash 省钱。

---

## 九、安全注意事项

1. **API Key**：建议用 SecretRef 环境变量引用替代明文：`"apiKey": { "source": "env", "provider": "default", "id": "DEEPSEEK_API_KEY" }`
2. **Gateway**：bind=loopback 时仅本地可访问，不暴露公网
3. **防火墙**：云服务器安全组建议只开放必要端口
4. **定期检查**：`openclaw doctor`、`openclaw security audit`
5. **备份配置**：`~/.openclaw/openclaw.json` 和 `~/.openclaw/workspace/` 是核心文件

---

## 十、文件清单（部署后检查）

```
/root/.openclaw/
├── openclaw.json          # 核心配置
├── workspace/
│   ├── IDENTITY.md        # AI 身份 ← 必须填写
│   ├── USER.md            # 用户信息 ← 必须填写
│   ├── SOUL.md            # 行为准则
│   ├── AGENTS.md          # 工作空间说明
│   ├── TOOLS.md           # 本地工具笔记
│   ├── MEMORY.md          # 长期精选记忆 ← 必须创建
│   ├── BOOTSTRAP.md       # 引导脚本 ← 完成后删除
│   └── memory/
│       └── YYYY-MM-DD.md  # 每日记忆 ← 自动积累
```
