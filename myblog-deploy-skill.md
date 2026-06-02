# 个人博客部署 Skill — 腾讯云服务器共存

> 在已运行 OpenClaw 微信机器人的同一台腾讯云服务器上部署个人博客（React + Express）
> 更新：2026-06-02
> 环境：腾讯云轻量应用服务器 4核4G (OpenCloudOS 9.4) + OpenClaw 已运行

---

## 一、项目概况

### 1.1 博客技术栈
| 层 | 技术 |
|----|------|
| 前端 | React 19 + TypeScript + Vite 8 + Tailwind CSS 4 |
| 后端 | Express 5 (Node.js)，JSON 文件存储 |
| AI 聊天 | DeepSeek API 兼容（流式输出），构建时注入前端 |
| 功能 | 日记、相册、收藏、动漫、关于页 |

### 1.2 服务共存架构

```
                  腾讯云轻量服务器 (4核4G 3Mbps)
                  ┌─────────────────────────────────┐
用户浏览器          │                                 │
────────→ :80     │  Nginx (systemd)                │
                  │  ├── /          → 静态文件      │
                  │  ├── /api/*     → 127.0.0.1:3001│
                  │  └── /assets/*  → 30天缓存      │
                  │                                 │
用户微信            │  OpenClaw Gateway (crontab)     │
────────→ WeChat  │  └── 127.0.0.1:18789           │
                  │                                 │
                  │  PM2: myblog-api (Express)       │
                  │  数据: /opt/myblog/data/*.json   │
                  └─────────────────────────────────┘
```

**资源占用**：博客 Express ~70MB + Nginx ~20MB + OpenClaw ~200MB ≈ 300MB，4GB 内存仅用 7%。

---

## 二、本地构建（Windows）

### 2.1 前置条件
- Node.js ≥ 22（通过 nvm 安装）
- 项目源码位于本机

### 2.2 创建 .env（AI 聊天密钥）

```bash
cd ~/Desktop/myblog
cp .env.example .env
```

编辑 `.env`（**必须在构建前完成**——`VITE_*` 变量在 `vite build` 时硬编码进 JS）：

```env
VITE_OPENAI_API_KEY=sk-你的DeepSeek密钥
VITE_OPENAI_BASE_URL=https://api.deepseek.com
VITE_OPENAI_MODEL=deepseek-v4-pro
```

### 2.3 构建

```bash
npm install
npm run build
```

产出 `dist/` 目录，含 `index.html` + `assets/`（JS ~184KB gzipped，CSS ~8KB gzipped）。

---

## 三、服务器部署

### 3.1 SSH 密钥配置（一次性）

**客户端**生成并复制公钥：

```bash
ssh-keygen -t ed25519 -C "openclaw-deploy" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

**服务器端**（腾讯云 Web 控制台 / VNC 登录）：

```bash
cat << 'EOF' > /root/.ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... openclaw-deploy
EOF
chmod 600 /root/.ssh/authorized_keys
```

### 3.2 打包上传

```bash
cd ~/Desktop/myblog

# 打包（排除 node_modules 和源码）
tar -czf /tmp/myblog-deploy.tar.gz \
  --exclude='node_modules' \
  --exclude='.git' \
  --exclude='deploy' \
  --exclude='src' \
  --exclude='public' \
  dist/ server/ package.json package-lock.json data/ .env.example

# 上传
scp -i ~/.ssh/id_ed25519 /tmp/myblog-deploy.tar.gz root@<服务器IP>:/tmp/
```

### 3.3 服务器端安装

```bash
ssh -i ~/.ssh/id_ed25519 root@<服务器IP>
```

```bash
# 加载 nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"

# 解压
mkdir -p /opt/myblog /var/log/myblog
cd /opt/myblog
tar -xzf /tmp/myblog-deploy.tar.gz
rm /tmp/myblog-deploy.tar.gz

# 安装生产依赖
npm install --omit=dev
```

### 3.4 安装配置 Nginx

```bash
dnf install -y nginx
```

写入 `/etc/nginx/conf.d/blog.conf`：

```nginx
server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/blog-access.log;
    error_log  /var/log/nginx/blog-error.log;

    client_max_body_size 50m;

    root /opt/myblog/dist;
    index index.html;

    # API 反向代理
    location /api/ {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;        # SSE 流式输出支持
        proxy_cache off;
        proxy_read_timeout 300s;
    }

    # SPA 路由回退
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 静态资源长期缓存
    location /assets/ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # gzip 压缩（节省 3Mbps 带宽）
    gzip on;
    gzip_types text/plain text/css application/json application/javascript
             text/xml application/xml text/javascript image/svg+xml;
    gzip_min_length 256;
    gzip_vary on;
}
```

```bash
nginx -t                          # 测试配置
systemctl enable nginx
systemctl start nginx
```

### 3.5 安装 PM2 并启动博客

```bash
npm install -g pm2
```

写入 `/opt/myblog/ecosystem.config.cjs`：

```js
module.exports = {
  apps: [{
    name: 'myblog-api',
    script: 'server/index.js',
    cwd: '/opt/myblog',
    env: {
      NODE_ENV: 'production',
      API_PORT: '3001',
    },
    out_file: '/var/log/myblog/out.log',
    error_file: '/var/log/myblog/err.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    max_memory_restart: '300M',
    autorestart: true,
  }],
};
```

```bash
cd /opt/myblog
pm2 start ecosystem.config.cjs
pm2 save
pm2 startup systemd -u root --hp /root
```

---

## 四、开放防火墙

**腾讯云控制台** → 轻量应用服务器 → 防火墙 → 添加规则：

| 应用类型 | 协议 | 端口 | 来源 |
|---------|------|------|------|
| HTTP    | TCP  | 80   | 0.0.0.0/0 |

> ⚠️ OpenClaw 端口 18789 是 `bind: loopback`，**不要**开放公网！

---

## 五、验证

```bash
# 端口检查
ss -tlnp | grep -E "80|3001|18789"
# 预期：80(nginx) 3001(node) 18789(openclaw) 全部 LISTEN

# API 测试
curl -s http://127.0.0.1:3001/api/about | python3 -m json.tool | head -5

# 前端测试
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/
# 预期：200

# 进程状态
pm2 status              # myblog-api: online
systemctl is-active nginx   # active
pgrep -f "openclaw gateway" # 有 PID 输出
```

浏览器访问 `http://<服务器IP>` 即可看到博客。

---

## 六、运维命令

```bash
# === 博客 ===
pm2 status                  # 进程状态
pm2 logs myblog-api         # 实时日志（Ctrl+C 退出）
pm2 restart myblog-api      # 重启
pm2 stop myblog-api         # 停止

# === Nginx ===
systemctl reload nginx      # 重载配置（不中断服务）
systemctl restart nginx     # 重启
tail -f /var/log/nginx/blog-access.log   # 访问日志

# === OpenClaw（保持原方式）===
tail -f /tmp/openclaw/gateway.log
kill -9 $(pgrep -f "openclaw gateway")   # 停止
# 重启用 crontab 里配的启动命令

# === 磁盘 ===
df -h                       # 磁盘使用（40GB SSD）
free -h                     # 内存使用
du -sh /opt/myblog/*        # 博客占用
```

---

## 七、更新博客

```bash
# 本地
cd ~/Desktop/myblog
npm run build
tar -czf /tmp/myblog-update.tar.gz dist/ server/ package.json
scp -i ~/.ssh/id_ed25519 /tmp/myblog-update.tar.gz root@<IP>:/tmp/

# 服务器
ssh -i ~/.ssh/id_ed25519 root@<IP>
cd /opt/myblog
tar -xzf /tmp/myblog-update.tar.gz
npm install --omit=dev          # 如有新依赖
pm2 restart myblog-api          # 重启 API
systemctl reload nginx          # 如 nginx 配置有变
```

---

## 八、常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 浏览器访问不了 | 防火墙未开放 80 端口 | 腾讯云控制台 → 防火墙 → 添加 TCP:80 |
| 502 Bad Gateway | Express 没启动 | `pm2 restart myblog-api` |
| AI 聊天不工作 | 构建时 `.env` 未配置或 API Key 错误 | 本地创建 `.env`，重新 `npm run build` 部署 |
| 日记/相册数据丢失 | data/ 目录未被备份 | 数据在 `/opt/myblog/data/*.json`，定期 rsync 备份 |
| Nginx 启动报错 | 配置语法错误 | `nginx -t` 查看具体行号 |
| 端口 3001 冲突 | 被其他进程占用 | `ss -tlnp \| grep 3001` 查占用，改 `API_PORT` |
| 和 OpenClaw 冲突 | 不可能——不同端口 | 博客 80/3001，OpenClaw 18789，互不干扰 |
| SSH Permission denied | 密钥未添加或权限错误 | `.ssh/` 700，`authorized_keys` 600 |
| Node.js 未找到 | SSH 非交互式不加载 nvm | 脚本开头加 `export NVM_DIR` + `source nvm.sh` |

---

## 九、安全加固（可选）

### 9.1 AI Key 后端代理

当前方案 `VITE_OPENAI_API_KEY` 构建时嵌入前端 JS，浏览器可直接看到。如需加固，在 `server/index.js` 添加代理：

```js
// 追加到 server/index.js
app.post('/api/chat/proxy', async (req, res) => {
  const DEEPSEEK_KEY = process.env.DEEPSEEK_API_KEY;
  const upstream = await fetch('https://api.deepseek.com/chat/completions', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${DEEPSEEK_KEY}`,
    },
    body: JSON.stringify(req.body),
  });
  res.setHeader('Content-Type', 'text/event-stream');
  upstream.body.pipe(res);
});
```

前端 `src/services/openai.ts` 改调 `/api/chat/proxy`，删除 `.env` 中的 `VITE_OPENAI_API_KEY`。

### 9.2 HTTPS（有域名后）

```bash
dnf install -y certbot python3-certbot-nginx
certbot --nginx -d your-domain.com
# 证书自动续期
```

---

## 十、文件清单（部署后检查）

```
服务器端：
/opt/myblog/
├── dist/                  # 前端静态文件（Nginx root）
├── server/index.js        # Express API（PM2 运行）
├── data/                  # JSON 数据文件（日记、相册等）
│   ├── diaries.json
│   ├── albums.json
│   ├── photos.json
│   ├── bookmarks.json
│   ├── anime.json
│   └── about.json
├── node_modules/          # 生产依赖
├── package.json
├── ecosystem.config.cjs   # PM2 配置
└── .env                   # 环境变量（可选，Express 用）

/etc/nginx/conf.d/blog.conf   # Nginx 配置
/var/log/myblog/              # PM2 日志
/var/log/nginx/               # Nginx 日志
```

---

## 十一、一键部署脚本

项目中的 `deploy/deploy.sh` 封装了以上全部步骤：

```bash
bash deploy/deploy.sh <服务器IP>
```

自动完成：构建 → 打包 → 上传 → 解压安装 → Nginx 配置 → PM2 启动。
