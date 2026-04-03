# DeerFlow Windows 本地启动指南

本文档介绍如何在 Windows 系统上不使用 Docker 和 Nginx 直接启动 DeerFlow 项目。
# [DeerFlow 原README](./README-orgin.md)

## 前置条件

- **Python 3.12+**
- **uv** (Python 包管理器) - 安装: `pip install uv`
- **Node.js 22+**
- **pnpm** - 安装: `npm install -g pnpm`
- **mingw32-make** (可选，用于替代 make 命令) - 安装 MinGW 后可用

## 快速启动

### 1. 配置文件准备

使用 `mingw32-make` 代替 `make`:

```bash
mingw32-make config
```

或者手动复制配置文件:

```powershell
copy config.example.yaml config.yaml
copy .env.example .env
copy frontend\.env.example frontend\.env
```

### 2. 配置 API 密钥

编辑 `.env` 文件，设置必要的 API 密钥:

```bash
# 模型 API 密钥 (至少配置一个)
OPENAI_API_KEY=your-openai-api-key
# 或其他模型
VOLCENGINE_API_KEY=your-volcengine-api-key
DEEPSEEK_API_KEY=your-deepseek-api-key

# 搜索工具 API 密钥
TAVILY_API_KEY=your-tavily-api-key
JINA_API_KEY=your-jina-api-key

# CORS 配置 (允许前端跨域访问)
CORS_ORIGINS=http://localhost:3000,http://localhost:3001,http://localhost:2026
```

### 3. 配置前端直接连接后端

编辑 `frontend\.env` 文件，修改为直接连接后端服务:

```bash
# 直接连接后端服务（不使用 Nginx 代理）
NEXT_PUBLIC_BACKEND_BASE_URL="http://localhost:8001"
NEXT_PUBLIC_LANGGRAPH_BASE_URL="http://localhost:2024"
```

### 4. 配置模型

编辑 `config.yaml` 文件，配置至少一个模型:

```yaml
models:
  - name: gpt-4o
    display_name: GPT-4o
    use: langchain_openai:ChatOpenAI
    model: gpt-4o
    api_key: $OPENAI_API_KEY
    supports_vision: true

  # 或使用火山引擎模型
  - name: doubao-seed-1.8
    display_name: Doubao-Seed-1.8
    use: deerflow.models.patched_deepseek:PatchedChatDeepSeek
    model: doubao-seed-1-8-251228
    api_base: https://ark.cn-beijing.volces.com/api/v3
    api_key: $VOLCENGINE_API_KEY
    supports_thinking: true
    supports_vision: true
```

## 启动服务

需要开启 **3 个终端** 分别启动服务。

### 终端 1 - LangGraph Server (端口 2024)

```powershell
cd backend
uv sync
uv run langgraph dev --no-browser --allow-blocking
```

### 终端 2 - Gateway API (端口 8001)

```powershell
cd backend
uv run uvicorn app.gateway.app:app --host 0.0.0.0 --port 8001 --reload
```

### 终端 3 - Frontend (端口 3000)

```powershell
cd frontend
pnpm install
pnpm dev
```

## 访问应用

启动完成后，打开浏览器访问: **http://localhost:3000**

## 服务端口说明

| 服务 | 端口 | 用途 |
|------|------|------|
| LangGraph Server | 2024 | Agent 核心，处理对话、工具调用、子代理 |
| Gateway API | 8001 | REST API，管理模型、MCP、Skills、Memory |
| Frontend | 3000 | Web 界面 |

## 常见问题

### CORS 跨域错误

如果出现 `CORS policy` 错误，确保:

1. `.env` 文件中配置了 `CORS_ORIGINS=http://localhost:3000`
2. Gateway API 已添加 CORS 中间件 (已在 `backend/app/gateway/app.py` 中配置)
3. 重启 Gateway API 服务

### 前端连接 2026 端口

如果前端仍然尝试连接 `localhost:2026`，检查:

1. `frontend\.env` 文件是否正确配置
2. **重启前端服务** (Next.js 只在启动时读取环境变量)

### 模型配置错误

如果 LangGraph 启动失败，检查:

1. `config.yaml` 中至少配置了一个有效模型
2. 对应的 API 密钥已在 `.env` 中设置

## 与 Docker/Nginx 模式的区别

| 特性 | Docker + Nginx | 本地直接启动 |
|------|---------------|-------------|
| 访问地址 | http://localhost:2026 | http://localhost:3000 |
| API 代理 | Nginx 统一代理 | 前端直接连接后端 |
| 跨域处理 | Nginx 处理 | FastAPI CORS 中间件 |
| 沙箱隔离 | Docker 容器 | 本地执行 |
| 适用场景 | 生产环境 | 开发调试 |

## 注意事项

1. **Sandbox 模式**: 默认使用 `LocalSandboxProvider`，代码直接在本机执行
2. **数据持久化**: 会话状态保存在 `backend/checkpoints.db` (SQLite)
3. **日志位置**: 各服务日志输出到各自终端
