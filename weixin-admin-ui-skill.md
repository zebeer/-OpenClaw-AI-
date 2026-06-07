# 微信伴侣管理后台 Skill

> 为 OpenClaw 微信 AI 伴侣构建可视化管理界面，集成到现有博客 /admin 区域
> 更新：2026-06-07
> 环境：Windows 11 本地开发 + 腾讯云轻量应用服务器 (OpenCloudOS 9.4)

---

## 一、功能概览

基于博客现有 Win98 复古风格，在 `/admin/weixin` 路径下新增 7 个管理标签页：

| 标签 | 功能 |
|------|------|
| 仪表盘 | Gateway 状态、Bot 在线状态、记忆统计、服务器资源、Bootstrap 检查 |
| Bot管理 | Bot 在线状态、扫码登录（返回终端二维码）、登出 |
| 记忆 | 记忆文件列表+Markdown编辑、全文搜索、手动触发 Dreaming |
| 身份塑造 | AI 名字/Emoji/性格特质、说话风格滑条、行为准则配置 → 自动生成 IDENTITY.md |
| 配置 | openclaw.json 编辑器+JSON校验+备份恢复 |
| 日志 | CRT 终端风格实时日志查看、级别筛选、关键词搜索、导出 |
| 运维 | 重启/停止 Gateway、端口监控、openclaw doctor 健康检查 |

---

## 二、技术架构

```
浏览器 /admin/weixin
    ↓
React 页面 (WeixinAdminPage + 7 子组件)
    ↓ REST API
Express /api/admin/weixin/* (server/weixin-admin.js)
    ↓ child_process / fs
服务器 OpenClaw Gateway + ~/.openclaw/workspace/ 文件
```

### 复用现有技术栈
- React 19 + TypeScript + Vite 8 + Tailwind CSS 4
- Zustand + React Router 6 + Framer Motion + Lucide Icons
- Express 5 + 自定义 session + bcrypt
- Win98 复古设计语言 (win-titlebar, win-btn, Card 等)

---

## 三、文件清单

### 新增文件 (8个)
```
server/weixin-admin.js              # 后端 API (20+ 端点)
src/pages/WeixinAdminPage.tsx        # 主容器 + 7 Tab 导航
src/components/weixin/
├── WeixinDashboard.tsx              # 仪表盘
├── WeixinBotManager.tsx             # Bot 管理
├── WeixinMemoryManager.tsx          # 记忆管理
├── WeixinIdentityBuilder.tsx        # 身份与说话方式塑造 ⭐
├── WeixinConfigEditor.tsx           # 配置编辑器
├── WeixinLogViewer.tsx              # 日志查看器
└── WeixinOperations.tsx             # 运维操作
```

### 修改文件 (4个)
```
src/types/index.ts                   # +15 接口类型
src/services/api.ts                  # +api.weixin 命名空间
src/App.tsx                          # +路由 /admin/weixin
src/components/layout/Header.tsx     # +导航入口
server/index.js                      # +引入 weixin-admin 模块
```

---

## 四、后端 API 端点

所有端点需要 `requireAdmin` 权限，挂载在 `/api/admin/weixin/*`：

### 状态
```
GET  /status              → Gateway+Boot+记忆+服务器+Bot+端口
GET  /bot/status           → Bot 在线状态、插件列表
GET  /doctor               → openclaw status + doctor 输出
GET  /ports                → 80/3001/18789 端口占用
```

### Bot 操作
```
POST /bot/login            → 执行扫码登录，返回终端二维码输出
POST /bot/logout           → 登出微信 Bot
```

### 记忆管理
```
GET  /memory/list          → 记忆文件列表 (名称/大小/日期)
GET  /memory/file?name=    → 读取指定文件内容
PUT  /memory/file          → 保存文件 { name, content }
GET  /memory/search?q=     → 全文搜索
POST /memory/dream         → 手动触发 Dreaming
```

### 身份文件
```
GET  /identity/:file       → 读取 IDENTITY.md/USER.md/SOUL.md
PUT  /identity/:file       → 保存身份文件 { content }
```

### 配置管理
```
GET  /config               → 读取 openclaw.json + 解析验证
PUT  /config               → 保存 (自动校验 JSON 语法)
POST /config/validate      → 单独校验 JSON
POST /config/backup        → 创建备份
GET  /config/backups       → 列出所有备份
POST /config/restore       → 从指定备份恢复
```

### 日志
```
GET  /logs?level=&search=&lines=  → 获取日志 (支持级别/搜索/行数)
GET  /logs/stream                 → SSE 实时日志流
```

### 运维
```
POST /gateway/restart      → 重启 Gateway (kill + nohup)
POST /gateway/stop         → 停止 Gateway
```

---

## 五、身份塑造器详解

这是核心特色功能，提供**可视化配置面板**替代手写 Markdown 文件。

### 三个子配置页

**A. 基础身份 → IDENTITY.md**
- AI 名字输入框
- Emoji 选择器（16 个预设：🌿🌸⭐🌙🦋🐱🐰🦊🐼🎀💫🍀🎵💜✨🔥）
- 性格特质 Tag 多选（20 个：温柔/安静/俏皮/幽默/理性/毒舌/元气/慵懒/傲娇/天然呆...）
- 自我认知 Markdown 文本区

**B. 说话风格 → 追加到 IDENTITY.md**
- 语气滑条 ×3：正式↔随意、温暖↔冷静、简洁↔详细
- 回复长度：短句 / 中等 / 长篇
- Emoji 使用频率：从不 / 偶尔 / 经常 / 狂熱
- 语言风格：简体中文 / 繁体中文 / 中英混合
- 自称方式 / 对用户的称呼

**C. 行为准则 → SOUL.md 补充**
- 角色定位：朋友 / 助手 / 伴侣 / 树洞 / 老师 / 自定义
- 主动性：仅回复 / 偶尔主动 / 经常关心
- 话题偏好/回避 Tag 多选
- 场景模板：日常问候、用户难过时、用户开心时

底部可预览生成的 IDENTITY.md 和 SOUL 补充内容，确认后一键保存。

---

## 六、部署流程

### 6.1 本地构建
```bash
cd ~/Desktop/myblog
npm run build
# 输出: tsc -b && vite build → dist/
```

### 6.2 打包上传
```bash
cd ~/Desktop/myblog
tar -czf /tmp/myblog-deploy.tar.gz \
  --exclude='node_modules' \
  --exclude='.git' \
  --exclude='src' \
  --exclude='public' \
  --exclude='data' \
  --exclude='deploy' \
  dist/ server/ package.json package-lock.json

scp -i ~/.ssh/id_ed25519 /tmp/myblog-deploy.tar.gz root@<服务器IP>:/tmp/
```

### 6.3 服务器安装
```bash
ssh root@<服务器IP>

# 加载 nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"

# 解压替换
cd /opt/myblog
tar -xzf /tmp/myblog-deploy.tar.gz
rm /tmp/myblog-deploy.tar.gz

# 安装依赖（无新增依赖时可跳过）
npm install --omit=dev

# 重启
pm2 restart myblog-api
```

### 6.4 验证
```bash
pm2 status                    # myblog-api: online
curl -s http://127.0.0.1:3001/api/admin/weixin/status | head -c 200
```

浏览器访问 `http://<服务器IP>/admin/weixin` → 管理员登录 → 管理后台。

---

## 七、环境适配

后端自动检测运行平台：

| 项目 | Windows (开发) | Linux (生产) |
|------|---------------|-------------|
| 工作区路径 | `~/.openclaw/workspace/` | `/root/.openclaw/workspace/` |
| 配置路径 | `~/.openclaw/openclaw.json` | `/root/.openclaw/openclaw.json` |
| 日志路径 | N/A | `/tmp/openclaw/gateway.log` |
| 运维操作 | 返回提示（仅Linux可用） | 完整支持 |
| openclaw 命令 | `openclaw` | `/root/.nvm/.../bin/openclaw` |

---

## 八、安全注意事项

1. **所有 API 端点** 需要 `requireAdmin` 认证（管理员账号登录）
2. **路径安全**：文件名使用 `basename()` 过滤，防止路径遍历
3. **JSON 校验**：配置保存前强制校验 JSON 语法，防止写坏配置
4. **备份机制**：配置修改前可选备份，恢复时先备份当前配置
5. **运维操作**：重启/停止需要二次确认弹窗
6. **Gateway bind=loopback**：端口 18789 不暴露公网

---

## 九、常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 仪表盘显示 Gateway 离线 | Gateway 未启动或已崩溃 | 前往「运维」→ 重启 Gateway |
| Bot 状态显示"未配置" | 微信插件未安装 | SSH 执行 `openclaw plugins install "@tencent-weixin/openclaw-weixin"` |
| 扫码登录无二维码 | 命令在服务器终端执行 | 查看「Bot管理」→ 登录输出的文本中可能包含终端二维码 |
| 记忆文件列表为空 | 未完成 Bootstrap | 前往「身份塑造」完成身份配置 |
| 身份保存后 AI 没变化 | Gateway 需重启才能加载新文件 | 前往「运维」→ 重启 Gateway |
| 配置保存报 JSON 错误 | JSON 格式问题 | 点击「校验 JSON」查看具体错误行 |
| 日志显示"文件不存在" | Gateway 未在产生日志 | 先启动 Gateway |
| 本地开发看不到运维操作 | Windows 不支持 | 部署到 Linux 服务器后可用 |

---

## 十、后续可扩展

- 实时二维码在浏览器端显示（需要将终端 QR 转为图片）
- 多 Bot 账号管理
- 聊天记录查看与搜索
- 记忆分析统计（情感趋势、话题分布）
- 自动化 Bootstrap 向导（引导式填写身份）
- WebSocket 实时日志推送替代轮询
- 移动端适配
