# 实施计划：部署 Open WebUI

## 前置条件检查

已确认：
- ✅ uv 已安装 (v0.9.16)
- ✅ Ollama 已安装（用户确认）
- ⚠️ Python 3.11 未安装（当前系统 Python 3.13.1）
- ⚠️ fnm 未在 Git Bash 中可用（需要在 PowerShell 中操作）
- ⚠️ Node.js 当前版本 v24，需要切换到 v21

## 实施步骤

### Phase 1: 环境准备

#### 1.1 安装 Python 3.11

**方案：使用 uv 的 Python 管理功能**

uv 0.8+ 版本支持自动下载和管理 Python 版本，无需手动安装。

```powershell
# 在 PowerShell 中执行
cd D:\D_Code\open-webui\backend

# uv 会自动下载 Python 3.11
uv venv --python 3.11

# 验证
.\.venv\Scripts\Activate.ps1
python --version  # 应显示 Python 3.11.x
```

**备选方案**（如果 uv 自动下载失败）：
1. 从 python.org 下载 Python 3.11.x Windows installer
2. 安装时勾选 "Add to PATH"
3. 使用 `uv venv --python 3.11` 创建虚拟环境

#### 1.2 安装和配置 Node.js 21

**方案：使用 fnm 在 PowerShell 中操作**

根据 `jianhui/node22安装排查记录.md` 的经验：

```powershell
# 1. 检查 fnm 是否已安装
fnm --version

# 如果未安装，使用 winget 安装
winget install Schniz.fnm

# 关闭并重新打开 PowerShell

# 2. 配置 PowerShell Profile（一次性操作）
# 检查 Profile 是否存在
Test-Path $PROFILE

# 如果不存在，先创建
if (!(Test-Path $PROFILE)) { New-Item -Path $PROFILE -ItemType File -Force }

# 添加 fnm 初始化（带防御性检查）
Add-Content $PROFILE "`nif (Get-Command fnm -ErrorAction SilentlyContinue) {`n    fnm env --use-on-cd --shell powershell | Out-String | Invoke-Expression`n}"

# 重新加载 Profile
. $PROFILE

# 3. 安装 Node.js 21（使用淘宝镜像加速）
$env:FNM_NODE_DIST_MIRROR = "https://npmmirror.com/mirrors/node/"
fnm install 21
fnm default 21
fnm use 21

# 验证
node -v  # 应显示 v21.x.x
npm -v
```

**重要提示**：
- 必须在 PowerShell 中操作，Git Bash 不支持 fnm
- 如遇到 "fnm not recognized"，完全退出并重新打开 PowerShell
- Profile 写入后必须重新加载或开新窗口才生效

#### 1.3 验证 Ollama

```powershell
# 检查 Ollama 版本
ollama --version

# 检查 Ollama 服务是否运行
curl http://localhost:11434/api/tags

# 如果 qwen3:4b 未下载，执行：
ollama run qwen3:4b
# 首次运行会下载模型（约 2.3GB）
```

### Phase 2: 后端部署

```powershell
# 切换到后端目录
cd D:\D_Code\open-webui\backend

# 2.1 创建并激活虚拟环境
uv venv --python 3.11
.\.venv\Scripts\Activate.ps1

# 验证 Python 版本
python --version  # 确保是 3.11.x

# 2.2 复制环境变量模板
if (!(Test-Path .env)) {
    Copy-Item ..\.env.example .env -ErrorAction SilentlyContinue
}

# 2.3 生成 WEBUI_SECRET_KEY 并写入 .env
$key = -join ((48..57)+(65..90)+(97..122) | Get-Random -Count 64 | ForEach-Object { [char]$_ })
if (Test-Path .env) { 
    Add-Content .env "`nWEBUI_SECRET_KEY=$key" 
} else { 
    Set-Content .env "WEBUI_SECRET_KEY=$key" 
}

# 2.4 设置 CORS（每次启动前需要设置）
$env:CORS_ALLOW_ORIGIN = "http://localhost:5173;http://localhost:8080"

# 2.5 安装依赖（使用 Tuna 镜像，约 340 个包）
# 镜像配置已在项目中完成
uv pip install -r requirements.txt

# 预计耗时：5-15 分钟（取决于网络）

# 2.6 启动后端
uvicorn open_webui.main:app --port 8080 --host 0.0.0.0 --ws-per-message-deflate true --reload

# 预期输出：
# - 首次启动需要 30-60 秒（加载大包）
# - 会创建 backend/data/ 目录和 SQLite 数据库
# - 会看到警告 "Frontend build directory not found" - 这是正常的
# - 成功标志：INFO: Uvicorn running on http://0.0.0.0:8080
```

**验证后端**（在新的 PowerShell 窗口）：
```powershell
# 健康检查
curl http://localhost:8080/health
# 应返回 {"status":true}

# 打开 Swagger 文档
start http://localhost:8080/docs
```

### Phase 3: 前端部署

```powershell
# 新开一个 PowerShell 窗口
cd D:\D_Code\open-webui

# 3.1 确认 Node.js 版本
node -v  # 应该是 v21.x.x

# 3.2 配置 npm 镜像（窗口级，关窗失效）
$env:npm_config_registry = "https://registry.npmmirror.com"

# 如果需要安装 Cypress，设置镜像（首次 npm install 时）
$env:CYPRESS_DOWNLOAD_MIRROR = "https://npmmirror.com/mirrors/cypress/"

# 3.3 安装依赖
npm install
# 预计耗时：3-10 分钟
# 可能出现的警告：
# - engine-strict: 如果 Node 版本不对会直接报错
# - peer dependencies: 可以忽略

# 3.4 启动前端开发服务器
npm run dev

# 预期输出：
# - VITE v5.x.x ready in xxx ms
# - Local: http://localhost:5173/
# - Network: http://192.168.x.x:5173/
```

**验证前端**：
```powershell
# 在浏览器中打开
start http://localhost:5173
```

### Phase 4: 首次配置

1. **访问应用**：打开 http://localhost:5173
2. **创建管理员账号**：
   - 首次访问会显示注册界面
   - 填写邮箱、用户名、密码
   - 点击创建账号
3. **配置 Ollama 连接**：
   - 登录后进入设置
   - 找到"连接"或"Models"部分
   - Ollama URL: `http://localhost:11434`
   - 测试连接
   - 选择 qwen3:4b 模型

## 验证清单

执行完成后逐项验证：

- [ ] Python 虚拟环境创建成功，版本为 3.11.x
- [ ] Node.js 版本切换到 v21.x.x
- [ ] Ollama 服务运行中，qwen3:4b 模型可用
- [ ] 后端依赖安装完成（无报错）
- [ ] 后端启动成功，能访问 /health 和 /docs
- [ ] 前端依赖安装完成（无报错）
- [ ] 前端启动成功，能访问 localhost:5173
- [ ] 能看到 Open WebUI 登录/注册界面
- [ ] 成功创建管理员账号并登录
- [ ] Ollama 连接配置成功
- [ ] 能够与 qwen3:4b 模型对话

## 常见问题与解决方案

### 问题 1：uv venv 无法下载 Python 3.11

**症状**：`uv venv --python 3.11` 报错找不到 Python 3.11

**解决**：
1. 手动安装 Python 3.11：https://www.python.org/downloads/
2. 安装时勾选 "Add Python to PATH"
3. 重启 PowerShell
4. 再次运行 `uv venv --python 3.11`

### 问题 2：fnm 命令找不到

**症状**：PowerShell 中执行 `fnm` 显示无法识别

**解决**：
1. 确认已用 `winget install Schniz.fnm` 安装
2. **完全退出** PowerShell（不是关标签页），重新打开
3. 验证：`fnm --version`
4. 如果还不行，检查 Profile：`Select-String -Path $PROFILE -Pattern 'fnm'`

### 问题 3：后端启动超过 1 分钟无响应

**症状**：运行 uvicorn 命令后长时间无输出

**原因**：正常现象，正在导入大型库（torch, transformers, chromadb）

**解决**：耐心等待，首次启动需要 30-60 秒

### 问题 4：端口被占用

**症状**：`[Errno 10048] address already in use`

**解决**：
```powershell
# 查看 8080 端口占用
netstat -ano | findstr :8080

# 如果有进程占用，记下 PID，然后：
taskkill /PID <PID> /F

# 或者换端口启动
$env:PORT = "8090"
uvicorn open_webui.main:app --port 8090 ...
```

### 问题 5：npm install 失败

**症状**：EBADENGINE 错误

**原因**：Node.js 版本不符合 package.json 要求

**解决**：
```powershell
# 确认当前版本
node -v

# 如果不是 v21，切换：
fnm use 21

# 清理后重试
Remove-Item node_modules -Recurse -Force -ErrorAction SilentlyContinue
npm install
```

### 问题 6：Ollama 连接失败

**症状**：前端配置 Ollama 时显示连接失败

**解决**：
```powershell
# 检查 Ollama 服务
curl http://localhost:11434/api/tags

# 如果失败，启动 Ollama（Windows 下通常是自动启动的服务）
# 检查系统托盘是否有 Ollama 图标

# 重新运行模型
ollama run qwen3:4b
```

## 下一步操作

部署成功后：
1. 熟悉 Open WebUI 界面
2. 尝试与 qwen3:4b 对话
3. 探索 RAG（知识库）功能
4. 查看管理面板
5. 阅读 `jianhui/` 目录下的其他文档了解更多细节

## 回滚计划

如果部署失败需要回滚：

```powershell
# 1. 停止服务（Ctrl+C）

# 2. 删除虚拟环境
Remove-Item backend\.venv -Recurse -Force

# 3. 删除 node_modules
Remove-Item node_modules -Recurse -Force

# 4. 删除生成的数据（可选）
Remove-Item backend\data -Recurse -Force
Remove-Item backend\.env
Remove-Item backend\.webui_secret_key

# 5. 切换回原 Node 版本（如需要）
fnm use system  # 切回 v24
```
