# my-openviking-bot

精简版 OpenViking 镜像（仅 server + console + bot），基于官方源码自动构建，支持自动跟随官方版本升级。

## 镜像地址

构建成功后自动推送到 GHCR：

```
ghcr.io/<你的GitHub用户名>/my-openviking-bot:<版本号>
ghcr.io/<你的GitHub用户名>/my-openviking-bot:latest
```

## 快速开始

### 1. 推送到 GitHub

```bash
cd /home/qh/agent-saas/openviking
git init
git add .
git commit -m "Add trimmed OpenViking build workflow"
git remote add origin https://github.com/<你的用户名>/my-openviking-bot.git
git push -u origin main
```

### 2. 启用 GitHub Actions

- 进入 GitHub 仓库 **Settings → Actions → General**
- **Workflow permissions** 选择 **Read and write permissions**
- 保存

### 3. 触发首次构建

- 进入 **Actions** 标签页
- 选择 **Build trimmed OpenViking (bot only)**
- 点击 **Run workflow**
- 填入版本号（如 `0.3.22`）或留空自动获取最新
- 点击 **Run workflow**

首次构建约 3-5 分钟，后续缓存命中约 30 秒。

### 4. 使用镜像

```bash
# 拉取镜像
docker pull ghcr.io/<你的用户名>/my-openviking-bot:latest

# 运行（配置分离挂载）
docker run -d \
  --name openviking \
  -p 1933:1933 -p 8020:8020 \
  -v ~/.openviking:/app/.openviking \
  ghcr.io/<你的用户名>/my-openviking-bot:latest
```

## 配置分离说明

镜像支持**完全分离配置**，通过挂载不同目录实现：

| 配置类型 | 容器内路径 | 宿主机挂载示例 | 说明 |
|---------|-----------|---------------|------|
| 主配置 | `/app/.openviking/ov.conf` | `-v ~/.openviking/ov.conf:/app/.openviking/ov.conf` | 服务端核心配置（模型、嵌入、存储等） |
| CLI配置 | `/app/.openviking/ovcli.conf` | `-v ~/.openviking/ovcli.conf:/app/.openviking/ovcli.conf` | CLI 工具配置 |
| 数据目录 | `/app/.openviking/data` | `-v ~/.openviking/data:/app/.openviking/data` | 向量索引、会话记忆、技能等持久化数据 |
| **整体挂载** | `/app/.openviking` | `-v ~/.openviking:/app/.openviking` | **推荐：一次性挂载整个目录** |

### 典型用法

```bash
# 推荐：整体挂载（最简单）
docker run -d \
  -v ~/.openviking:/app/.openviking \
  -p 1933:1933 -p 8020:8020 \
  ghcr.io/<用户名>/my-openviking-bot:latest

# 进阶：分离挂载（配置与数据分盘、只读配置等）
docker run -d \
  -v /data/openviking/config:/app/.openviking \
  -v /data/openviking/data:/app/.openviking/data \
  -p 1933:1933 -p 8020:8020 \
  ghcr.io/<用户名>/my-openviking-bot:latest
```

### 配置文件生成

首次运行若无配置，可通过环境变量注入或进入容器初始化：

```bash
# 方式1：环境变量注入完整 JSON 配置
docker run -d \
  -e OPENVIKING_CONF_CONTENT='{"embedding": {"provider": "volcengine", ...}}' \
  -v ~/.openviking:/app/.openviking \
  ...

# 方式2：进入容器交互式初始化
docker run -it --rm \
  -v ~/.openviking:/app/.openviking \
  ghcr.io/<用户名>/my-openviking-bot:latest \
  openviking-server init
```

## 自动升级机制

- **定时触发**：每天 04:00 自动检查官方最新 tag 并构建
- **手动触发**：GitHub Actions 页面 → Run workflow → 填版本号
- **版本同步**：构建参数 `OPENVIKING_VERSION` 直接同步官方 tag

## 镜像特性

| 特性 | 说明 |
|------|------|
| **仅包含** | server + console + bot（含 VikingBot） |
| **移除** | gemini extra、Web Studio、SDK 版本校验、Studio 完整性校验 |
| **基础镜像** | python:3.13-slim-trixie |
| **Python** | 3.13.13 (官方 slim 版本) |
| **架构** | linux/amd64（如需 arm64 在 workflow 中添加） |
| **构建缓存** | cargo/uv/ccache 全缓存，二次构建 ~30s |

## 版本对应关系

| 本镜像版本 | 对应官方版本 | 包含功能 |
|-----------|-------------|---------|
| 0.3.22+ | v0.3.22+ | MCP、异步API、ZIP扁平化、路径去重、ARM64等 |
| 0.3.9 | v0.2.6 级别 | 无上述新功能（即 zhangyq/openviking 现状） |

---

**维护**：仅需在 GitHub 仓库 Actions 页面点击 Run workflow 或等待定时任务，无需本地任何操作。