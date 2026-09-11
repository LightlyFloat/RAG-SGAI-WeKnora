# 上手学习 open-webui 项目并完成部署启动

## Goal

帮助新成员（lfloat）上手 open-webui 项目，理解项目架构和功能，并成功在本地部署启动应用。

## Background

Open WebUI 是一个自托管的 AI 平台，功能丰富且可扩展，支持 Ollama 和 OpenAI 兼容的 API。

**项目技术栈：**
- **前端**: SvelteKit (Svelte 5) + TypeScript + TailwindCSS
- **后端**: Python FastAPI + uvicorn
- **数据库**: SQLite (默认) / PostgreSQL
- **向量数据库**: ChromaDB (默认，嵌入式)
- **AI 集成**: Ollama, OpenAI API, LMStudio, GroqCloud 等

**主要目录结构：**
- `src/` - Svelte 前端代码
- `backend/open_webui/` - Python FastAPI 后端
  - `main.py` - FastAPI 主应用入口 (124KB)
  - `routers/` - API 路由
  - `models/` - 数据模型
  - `config.py` - 配置管理 (133KB)
- `jianhui/` - 同事 jianhui 的文档目录（在 dev-jianhui 分支）
- `backend/requirements.txt` - Python 依赖

## What I Already Know

### 1. 启动文档已找到（dev-jianhui 分支）

已成功切换到 `dev-jianhui` 分支并找到启动文档：
- `jianhui/open-webui-启动说明.md` - 主启动说明
- `jianhui/后端启动分析.md` - 详细的后端启动分析（含代码证据）
- `jianhui/运行说明.md` - 完整运行说明
- `jianhui/依赖安装说明.md` - 340 个依赖包详解
- `jianhui/依赖文件对比.md` - 依赖文件对比
- `jianhui/node22安装排查记录.md` - Node.js 22 安装问题记录
- `jianhui/会话记录-2026-09-02.md` 等 - 历史会话记录

### 2. 启动流程（基于 jianhui 文档）

**环境要求：**
- Python 3.11（推荐）
- Node.js 21（推荐，范围 19-21）
- Ollama + qwen3:4b 模型
- Windows 11 环境
- 需要 VPN（安装依赖时）

**后端启动步骤：**
```bash
cd backend
uv venv --python 3.11
.venv\Scripts\activate
uv pip install -r requirements.txt

# 首次运行需要设置环境变量
$env:CORS_ALLOW_ORIGIN = "http://localhost:5173;http://localhost:8080"

# 生成密钥（写入 backend/.env）
$key = -join ((48..57)+(65..90)+(97..122) | Get-Random -Count 64 | ForEach-Object { [char]$_ })
if (Test-Path .env) { Add-Content .env "`nWEBUI_SECRET_KEY=$key" } else { Set-Content .env "WEBUI_SECRET_KEY=$key" }

# 启动后端
uvicorn open_webui.main:app --port 8080 --host 0.0.0.0 --ws-per-message-deflate true --reload

# 验证
curl http://localhost:8080/health
```

**前端启动步骤：**
```bash
node -v  # 确保 >=19, <=21
$env:npm_config_registry = "https://registry.npmmirror.com"
npm install
npm run dev
```

### 3. 后端启动详解（来自后端启动分析文档）

**启动时序：**
1. 导入阶段（30-60秒，加载 torch/transformers/chromadb 等大包）
2. 数据库初始化（创建 `backend/data/`、SQLite `webui.db`、Alembic 迁移）
3. ChromaDB 初始化（嵌入式向量库，本地存储）
4. 预期警告：`Frontend build directory not found` - 正常，dev 模式前端由 Vite 提供
5. Ollama 探测（尝试连 localhost:11434）
6. 成功标志：`Uvicorn running on http://0.0.0.0:8080`

**关键知识点：**
- 启动时**不会**下载或加载 AI 模型
- SQLite 和 ChromaDB 都是嵌入式的，不需要额外服务
- RAG 功能首次使用时才下载 sentence-transformers 模型（90MB）
- 可通过 `HF_ENDPOINT=https://hf-mirror.com` 使用 HuggingFace 镜像

### 4. 当前环境检测结果

✅ **Node.js**: v24.18.0（超出推荐范围 19-21，可能有兼容性问题）  
⚠️ **Python**: 3.13.1（文档推荐 3.11，可能有兼容性问题）  
❌ **Ollama**: 未安装

## Requirements

1. **环境准备**
   - 解决 Python 版本问题（当前 3.13.1，推荐 3.11）
   - 解决 Node.js 版本问题（当前 24.18.0，推荐 21）
   - 安装 Ollama 并下载 qwen3:4b 模型

2. **后端部署**
   - 创建 Python 3.11 虚拟环境（使用 uv）
   - 安装后端依赖（340+ 包，使用 Tuna 镜像）
   - 配置环境变量和密钥
   - 启动后端服务并验证

3. **前端部署**
   - 安装前端依赖（使用 npm 镜像）
   - 启动前端开发服务器

4. **验证运行**
   - 后端健康检查：`http://localhost:8080/health`
   - 前端页面访问：`http://localhost:5173`
   - 创建管理员账号
   - 配置 Ollama 连接

## Acceptance Criteria

### Phase 1: 环境准备
- [x] 已切换到 dev-jianhui 分支
- [x] 已找到并阅读启动文档
- [x] 已检测当前环境版本
- [x] 已确认用户决策（Python 3.11, Node 22, Ollama 已安装）
- [x] 已创建实施计划（implement.md）
- [x] Python 3.11 虚拟环境创建成功
- [x] Node.js 切换到 v22.23.2（根据 jianhui 最新排查记录）
- [x] Ollama 服务验证通过，qwen3:4b 可用

### Phase 2: 后端部署
- [x] 后端依赖安装完成（340+ 包）
- [x] backend/.env 文件创建并配置密钥
- [x] 后端成功启动（Uvicorn running on 8080）
- [x] 后端健康检查通过（/health 返回正常）
- [x] Swagger 文档可访问（/docs）

### Phase 3: 前端部署
- [x] 前端依赖安装完成（npm install，1120 包）
- [x] 前端开发服务器启动成功（Vite running on 5173）
- [x] 能在浏览器访问 localhost:5173

### Phase 4: 功能验证
- [x] 看到 Open WebUI 注册/登录界面
- [x] 成功创建管理员账号
- [x] 成功登录系统
- [x] Ollama 连接配置成功
- [x] 能够与 qwen3:4b 模型正常对话

### Phase 5: 文档记录
- [x] 记录部署过程中遇到的问题和解决方案
- [x] 更新任务文档

## Open Questions

### 问题 1：Python 版本处理

**当前状态：** 您的系统 Python 是 3.13.1，但文档推荐 3.11

**选项 A（推荐）：** 使用 uv 创建 Python 3.11 虚拟环境
- 优点：完全隔离，符合文档要求
- 缺点：需要先安装 Python 3.11

**选项 B：** 尝试使用 Python 3.13.1
- 优点：快速开始，无需额外安装
- 缺点：可能遇到兼容性问题（FastAPI/Pydantic/依赖包）

**我的建议：** 选择 A，安装 Python 3.11 并使用 uv 创建虚拟环境。因为：
1. 文档经过验证，3.11 确定可用
2. 3.13 是较新版本，可能有依赖兼容性问题
3. 虚拟环境不影响系统 Python

**您希望如何处理？**

### 问题 2：Node.js 版本处理

**当前状态：** 您的 Node.js 是 v24.18.0，文档推荐 21，package.json 要求 >=18.13.0 <=22.x.x

**分析：**
- v24 超出了 package.json 的 engines 要求（<=22）
- 可能导致 npm install 警告或运行时问题

**选项 A：** 降级到 Node.js 21
- 使用 fnm/nvm 切换版本
- jianhui 文档中有 fnm 安装记录

**选项 B：** 先尝试 v24，遇到问题再降级
- 快速验证是否真的有兼容性问题

**我的建议：** 选择 B，先尝试 v24。因为：
1. 差异不大（24 vs 22），很多项目可以工作
2. 可以快速验证是否有实际问题
3. 如果失败，再用 fnm 降级到 21

**您希望如何处理？**

### 问题 3：Ollama 安装

**当前状态：** Ollama 未安装

**需要执行的步骤：**
1. 从 https://ollama.com 下载 Windows 安装包
2. 安装 Ollama
3. 运行 `ollama run qwen3:4b` 下载并启动模型（约 2.3GB）

**重要性：** 文档明确说明"启动后端需要安装好本地模型并运行 ollama"

**是否现在开始安装 Ollama？**

## Technical Notes

- 项目使用 Tuna 镜像源配置（已在项目中配置）
- uv 工具用于 Python 虚拟环境管理（比 venv 更快）
- npm 使用淘宝镜像：`https://registry.npmmirror.com`
- Cypress 下载需要 VPN
- 首次启动后端会创建 `backend/data/` 目录和 SQLite 数据库
- 首次使用 RAG 功能时会从 HuggingFace 下载模型（可配置镜像）
- 开发模式下后端不提供前端静态文件（这是正常的）
