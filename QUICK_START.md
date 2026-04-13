# TrendRadar 快速启动指南

## 🚀 最快启动方式 - Docker（推荐）

### 前置要求
- Docker & Docker Compose 已安装
- 配置文件就位

### 步骤 1: 准备配置文件

```bash
cd docker

# 编辑 .env 文件（填入必要配置）
# Windows: notepad .env
# Mac/Linux: nano .env
```

**必填项：**
- `WEBSERVER_PORT=8080` （可访问报告的端口）

**可选项（根据需要）：**
- 通知渠道：`FEISHU_WEBHOOK_URL`、`TELEGRAM_BOT_TOKEN` 等
- AI 功能：`AI_API_KEY`、`AI_MODEL`
- 远程存储：`S3_*` 系列

### 步骤 2: 启动容器

```bash
# 进入 docker 目录
cd docker

# 启动（会自动下载镜像，首次需要几分钟）
docker compose up -d

# 验证运行
docker compose ps
# 应该看到 trendradar 和 trendradar-mcp 都是 Up
```

### 步骤 3: 查看报告

打开浏览器访问：
```
http://localhost:8080
```

### 常用命令

```bash
# 查看日志
docker logs -f trendradar
docker logs -f trendradar-mcp

# 停止
docker compose down

# 重启
docker compose restart trendradar

# 手动执行爬取（触发一次完整流程）
docker exec trendradar python manage.py run

# 查看当前配置状态
docker exec trendradar python manage.py config

# 检查健康状态
docker exec trendradar python manage.py status
```

---

## 💻 本地运行（如果不用 Docker）

### 步骤 1: 安装依赖

**Windows:**
```bash
# 双击运行（自动检查 Python/UV，安装依赖）
setup-windows.bat
```

**Mac/Linux:**
```bash
bash setup-mac.sh
```

### 步骤 2: 验证配置

```bash
# 检查环境和配置是否正确
python -m trendradar --doctor
```

### 步骤 3: 运行

```bash
# 正常运行（一次完整流程）
python -m trendradar

# 查看调度状态（如果配置了 timeline.yaml）
python -m trendradar --show-schedule

# 测试通知渠道
python -m trendradar --test-notification
```

---

## 🔧 MCP Server 启动（用于 AI 分析）

### Docker 中自动启动

```bash
# 已在 docker-compose.yml 中配置
# 访问地址：http://127.0.0.1:3333
docker logs trendradar-mcp
```

### 本地启动

```bash
# HTTP 模式
bash start-http.sh          # Mac/Linux
# 或
start-http.bat              # Windows

# STDIO 模式（用于编辑器集成）
uv run python -m mcp_server.server
```

---

## 📋 配置快速参考

### 最小配置（仅热榜爬取）

编辑 `config/config.yaml`：
```yaml
app:
  timezone: "Asia/Shanghai"

platforms:
  enabled: true
  sources:
    - id: "baidu"      # 百度热搜
      name: "百度热搜"
    - id: "weibo"      # 微博
      name: "微博"

notification:
  enabled: false  # 先关通知，调试无误后再开

report:
  mode: "daily"   # 当日汇总模式
```

### 启用通知（选一个）

**飞书：**
```bash
# docker/.env
FEISHU_WEBHOOK_URL=https://open.feishu.cn/open-apis/bot/v2/hook/xxx
```

**Telegram：**
```bash
# docker/.env
TELEGRAM_BOT_TOKEN=1234567890:ABCDEFGHIJKLMNOPQRSTUVWxyz
TELEGRAM_CHAT_ID=987654321
```

**邮件：**
```bash
# docker/.env
EMAIL_FROM=your-email@gmail.com
EMAIL_PASSWORD=your-app-password
EMAIL_TO=recipient@example.com
EMAIL_SMTP_SERVER=smtp.gmail.com
EMAIL_SMTP_PORT=587
```

---

## 🆘 常见问题

### Q: Docker 无法启动

```bash
# 检查镜像是否存在
docker images | grep trendradar

# 如果没有，强制重新拉取
docker compose pull
docker compose up -d
```

### Q: 无法访问 http://localhost:8080

```bash
# 检查端口是否开放
netstat -ano | findstr :8080    # Windows
lsof -i :8080                   # Mac/Linux

# 如果被占用，改成其他端口
# 编辑 docker/.env
# WEBSERVER_PORT=8081
# 重启容器
docker compose restart
```

### Q: 报告为空

```bash
# 查看爬虫日志
docker logs trendradar

# 手动触发一次
docker exec trendradar python manage.py run

# 检查配置
python -m trendradar --doctor
```

### Q: 通知没收到

```bash
# 测试通知渠道
docker exec trendradar python manage.py test-notification

# 或本地
python -m trendradar --test-notification
```

---

## 📚 进阶操作

### 自定义关键词匹配

编辑 `config/frequency_words.txt`：
```txt
AI
ChatGPT

+技术          # 必须包含「技术」
AI

!广告          # 排除「广告」
```

### 启用 AI 分析

在 `docker/.env` 中填入：
```bash
AI_API_KEY=sk-xxx
AI_MODEL=deepseek/deepseek-chat
```

然后在 `config/config.yaml` 启用：
```yaml
ai_analysis:
  enabled: true
```

### 启用自动调度

创建 `config/timeline.yaml`（参考 README.md），设置按时间推送。

---

## ✅ 验证清单

- [ ] Docker/Docker Compose 已安装
- [ ] `config/config.yaml` 已存在
- [ ] `docker/.env` 已填入基本配置
- [ ] `docker compose up -d` 启动成功
- [ ] `docker compose ps` 显示两个容器都 Up
- [ ] 打开 http://localhost:8080 看到报告
- [ ] （可选）测试通知 `docker exec trendradar python manage.py test-notification`

---

## 🔗 后续

- 查看详细文档：`README.md`
- MCP 深度集成：`README-Cherry-Studio.md`
- 配置详解：`config/config.yaml` 内注释
- 问题排查：`python -m trendradar --doctor`
