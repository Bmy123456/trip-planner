# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

智能旅行助手 — 基于 HelloAgents 多智能体框架的 AI 旅行规划应用。用户输入目的地、日期和偏好后，四个专业智能体（景点搜索、天气查询、酒店推荐、行程规划）通过高德地图 MCP 工具协作，生成包含预算估算的结构化行程。

## 开发命令

### 后端（Python，在 `backend/` 目录下）

```bash
# 安装依赖
pip install -r requirements.txt

# 启动开发服务器（http://localhost:8000，自动重载）
python run.py

# API 文档：/docs（Swagger）和 /redoc
```

### 前端（Vue 3 + TypeScript，在 `frontend/` 目录下）

```bash
# 安装依赖
npm install

# 启动开发服务器（http://localhost:5173，/api 代理到 localhost:8000）
npm run dev

# 类型检查与构建
npm run build
```

本项目暂无测试套件。

## 架构

**后端**（`backend/app/`）：FastAPI 分层结构 — API 路由 → 服务 → 智能体。

- **agents/trip_planner_agent.py** — 核心多智能体编排器。创建四个 `SimpleAgent` 实例（景点、天气、酒店、规划），共享一个封装高德 MCP 服务器（`uvx amap-mcp-server`）的 `MCPTool`。规划按顺序执行：景点 → 天气 → 酒店 → 规划。规划智能体将三个结果综合为 JSON 行程，解析为 Pydantic 模型；解析失败时生成备用行程。
- **services/** — `llm_service.py`（HelloAgentsLLM 单例）、`amap_service.py`（直接调用高德 API 实现 POI/路线/搜索）、`unsplash_service.py`（景点图片）。
- **api/routes/** — 三个路由模块：`trip.py`（规划接口）、`poi.py`（POI 搜索）、`map.py`（路线+天气）。均挂载在 `/api` 下。
- **models/schemas.py** — 所有 Pydantic 请求/响应模型。主流程：`TripRequest` → `TripPlan`。
- **config.py** — 使用 pydantic-settings 从 `.env` 加载配置，同时检查 `../HelloAgents/.env` 获取 LLM 密钥。关键环境变量：`LLM_API_KEY`、`LLM_MODEL_ID`、`LLM_BASE_URL`、`AMAP_API_KEY`。

**前端**（`frontend/src/`）：Vue 3 Composition API + Ant Design Vue。

- 两个视图：`Home.vue`（规划表单）→ `Result.vue`（行程展示，含高德地图、导出图片/PDF）。
- `services/api.ts` — Axios 客户端，所有 API 调用集中在此。
- `types/index.ts` — TypeScript 接口，与后端 Pydantic 模型对应。

## 关键模式

- **MCP 工具调用**：智能体使用 `[TOOL_CALL:name:key=value,key=value]` 语法调用高德工具。MCPTool 的 `auto_expand=True` 将单个 MCP 服务器展开为多个可独立调用的函数。
- **单例服务**：`get_llm()`、`get_trip_planner_agent()` 等使用模块级全局变量实现懒加载单例。
- **Vite 代理**：前端开发服务器将 `/api` 代理到后端，开发时无跨域问题。
- **备用规划**：LLM JSON 解析失败时，`_create_fallback_plan` 生成通用占位行程，确保用户始终能得到响应。

## 必需的环境变量

`backend/.env` 至少需要配置：
- `LLM_API_KEY` / `LLM_BASE_URL` / `LLM_MODEL_ID` — LLM 服务提供商
- `AMAP_API_KEY` — 高德地图服务（后端 MCP 和前端 JS 地图均需要）

前端 `frontend/.env` 需要配置高德地图 Web JS Key。
