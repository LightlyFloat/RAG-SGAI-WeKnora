# 执行计划：WeKnora 部署与配置

前置阅读：本任务 `prd.md` 的 Decisions（尤其 D0）与 Confirmed Facts。
**D0 红线：不为 WeKnora 缺失的能力写任何自研代码。** 实施中若发现某项需求必须写代码
才能满足，先停下确认，不要自行扩大范围。

## 阶段一：仓库归置（R1 → AC1、AC2）

1. 全深度 clone 团队 fork 到独立目录：
   `git clone https://github.com/OrangeSunrise/WeKnora.git D:/D_Code/WeKnora`
2. 配置上游远端：
   `git remote add upstream https://github.com/Tencent/WeKnora.git`，
   随后 `git fetch upstream` 验证连通
3. 核对新 clone 的 HEAD 与 `VERSION`，确认仍为 v0.8.0 基线（F2 记录的 HEAD 为
   `1e36e07`；若上游已推进，记录实际 commit 以便复现）
4. **需用户确认后再执行**：删除 `D:/D_Code/open-webui/.temp-weknora/`。
   前置条件：阶段一第 1~3 步已完成且新 clone 可用。该目录为浅克隆且无本地改动，
   删除可逆（重新 clone 即可），但仍属 129MB 目录删除，执行前确认

## 阶段二：部署跑通（R2 → AC3）

5. 确认 Docker Desktop 已启用 WSL2 后端（F8：Windows 无原生路径），
   并确认可用内存满足 F10 估算
6. 由 `.env.example` 复制出 `.env`（**不要**用 `.env.lite.example`，见 D4）
7. 先完成阶段三的密钥替换，再首次启动 —— 避免用上游默认密钥初始化数据库后
   再换 `SYSTEM_AES_KEY` 导致已加密数据不可读（见风险点 RK1）
8. `docker compose up -d`，随后 `docker compose ps` 确认五服务健康：
   frontend、app、docreader、postgres（paradedb）、redis
9. 浏览器打开 Web UI，完成注册与登录

## 阶段三：密钥加固（R4 → AC5）

10. 替换 `.env` 中两项上游不安全默认值（F4）：
    - `JWT_SECRET`（默认 `weknora-jwt-secret`）
    - `SYSTEM_AES_KEY`（默认 `weknora-system-aes-key-32bytes!!`，需 32 字节）
11. 将 `SYSTEM_AES_KEY` 备份到团队约定的密钥保管位置，并在部署文档中记录
    **保管位置而非密钥本身**。丢失该密钥则所有已加密凭据不可恢复
12. 确认 `.env` 未被提交：WeKnora 的 `.gitignore:2` 以 `.*` 规则排除，
    执行 `git check-ignore -v .env` 验证

## 阶段四：模型接入（R3 → AC4）

13. 本地安装并启动 Ollama，拉取两个模型：
    - LLM：1.7b~4b 量化级别（对齐汇报 §7 的硬件约束）
    - Embedding：**必须 1024 维**，以对齐 migration `000059` 的
      HNSW/halfvec 索引（F11）。bge-m3 是该 migration 的目标模型
14. 在 WeKnora 设置页配置 LLM 与 Embedding 均指向本地 Ollama，逐项跑连通性测试
15. 确认 Embedding 维度显示为 1024
16. **不配置 rerank**（D5：WeKnora 无本地 rerank provider，按 D0 作废）

## 阶段五：最小验证（R5 → AC6、AC7）

17. 上传极小样本：数篇文档量级。**不要**上传 2000 页 S120 手册，
    也**不要**导入万级维基语料（D0）
18. 确认解析状态为完成、分块数大于 0
19. 就该文档提问，确认回答中出现引用标记
20. 点击引用，确认引用浮层与引用抽屉可打开并显示对应分块原文
21. 记录实测观感：分块级引用是否够用。这是 D2 后续复议页级引用的输入依据，
    **只记录不改造**

## 阶段六：文档沉淀（R6 → AC8）

22. 撰写部署文档，落在 open-webui 的 `document/`（D7），
    遵循 `.trellis/spec/guides/document-writing-style.md`：正式严谨、不口语化、
    不用解释性括号、保留全部技术信息
23. 按 `.trellis/spec/guides/project-doc-organization.md` 登记到 `document/文档索引.md`
24. 文档需覆盖阶段一至五全部步骤，使第二人可在干净环境复现 AC3~AC7

## 阶段七：交付发布与文档补齐

背景：阶段一至六的产出已落在 `D:/D_Code/WeKnora` 的 `dev-qingyang` 分支，提交
`20b84633`，工作树干净，尚未推送到任何远端。本阶段负责把成果发布到两个远端，
并补齐交付文档。

### 发布目标与内容边界

| 远端 | 仓库 | 分支 | 内容边界 |
| --- | --- | --- | --- |
| `origin` | OrangeSunrise/WeKnora | `dev-qingyang` | 仅交付内容，不含任何 AI 协作记录。新建远端分支，不发起合并请求 |
| `personal` | LightlyFloat/RAG-SGAI-WeKnora | `main` | 交付内容加 AI 协作记录，单分支，不设 dev 分支 |

### 执行步骤

25. 按下方「文档补齐清单」修订 `document/` 与 `.gitignore`，并新增
    `document/仓库与分支说明.md`
26. 将文档补齐结果提交到 `dev-qingyang`
27. 推送 `dev-qingyang` 到 `origin`。**前置条件**：LightlyFloat 需先获得
    OrangeSunrise/WeKnora 的 Write 协作者权限。2026-09-11 实测该权限为
    `pull: true, push: false`，`git push` 返回 HTTP 403，故本步骤在授权到位前
    无法执行
28. 由 `dev-qingyang` 建立本地分支 `personal-main`，新增 `ai-collab/` 目录归档
    Trellis 任务产物与 Claude 会话记录，提交后以
    `git push personal personal-main:main` 推送
29. 复核两个远端的内容边界：`origin` 的 `dev-qingyang` 不含 `ai-collab/`；
    `personal` 的 `main` 既含 `ai-collab/` 也含全部交付内容

### 文档补齐清单

1. `工作说明.md`：修正「仅修改 4 个配置文件」的口径。实测
   `git diff --name-status origin/main...HEAD` 为 6 个修改文件，除
   `builtin_agents.yaml`、`config.yaml`、`system_prompt.yaml`、`go.sum` 外
   还含 `.env.lite.example` 与 `.gitignore`。同时补充交付物清单指向 `delivery/`
2. `操作说明.md` 第六章：补齐知识库重建索引的完整流程，包括新建知识库的接口调用、
   旧知识库的整体删除方式、同一批文档触发重新解析的办法；写明更换语料时的落盘
   约定，即 `--dir` 可指向任意目录、`--ext` 默认仅收 `.md`、Lite 形态不支持 PDF；
   说明 `import_kb.py` 的 `--wait` 实际轮询全部知识库而非 `--kb` 指定的那一个；
   说明更换语料后 `manifest.csv` 的处理方式
3. `测试记录.md`：补入截图引用，现有截图位于 `document/` 下且未被该文档引用；
   补充执行人与执行日期；补充可复现的接口调用命令
4. `问题与解决.md`：新增未解决项汇总小节，新增统一的环境版本表，
   涵盖 mingw 16.1.0、SQLite 3.46.1、DuckDB v1.5.2 等散落在各表中的版本信息
5. `文档索引.md`：将「仓库中不含的两个文件」修正为四项，补入 `duckdb.dll` 与
   `source/*.zip`；登记 `文档索引.md` 自身、`document/` 下的截图、
   `source/wiki-kb/qdrant/collections/industrial_wiki/config.json`，并区分
   `07_evaluate_v2.py` 与 `retrieval_report_v2.md` 两个 v2 版本
6. `.gitignore` 第 72 行注释指向「`document/文档索引.md`「仓库中不含的文件」」，
   实际章节标题为「仓库中不含的两个文件」，需与修订后的标题一致
7. 新增 `document/仓库与分支说明.md`：说明三个远端的用途、两个发布目标的内容
   边界、后续如何把新工作同步到两个远端

### 安全决策记录

用户于 2026-09-11 明确确认：`LightlyFloat/RAG-SGAI-WeKnora` 保持 Public
可见性，Claude 会话记录原样推送，不做脱敏。已向用户说明该记录中包含
`.env.lite` 正在使用的 `SYSTEM_AES_KEY`、`TENANT_AES_KEY` 与 `JWT_SECRET`
三项活密钥，用户接受该风险。建议在推送完成后自行轮换这三项密钥，轮换后
需在界面重新录入模型凭据，已入库的知识库数据不受影响。

## 验证命令

| 目的 | 命令 |
| --- | --- |
| 确认全深度 clone（AC1） | `git rev-list --count HEAD`（应 > 1）、`ls .git/shallow`（应不存在） |
| 确认上游可拉取（AC1） | `git fetch upstream` |
| 确认 `.temp-weknora` 已清（AC2） | 在 open-webui 执行 `git status` |
| 确认服务健康（AC3） | `docker compose ps` |
| 排查启动失败 | `docker compose logs -f app`、`docker compose logs -f docreader` |
| 确认 `.env` 不入库 | `git check-ignore -v .env` |
| 确认对 origin 的写权限（步骤 27 前置） | `gh api repos/OrangeSunrise/WeKnora --jq '.permissions'`，`push` 需为 `true` |
| 确认 origin 分支已建立（AC 步骤 27） | `git ls-remote origin dev-qingyang` |
| 确认 origin 分支不含 AI 协作记录（步骤 29） | `git ls-tree -r --name-only origin/dev-qingyang \| Select-String ai-collab`，应无输出 |
| 确认 personal 的 main 含 AI 协作记录（步骤 29） | `git ls-tree -r --name-only personal/main \| Select-String ai-collab` |
| 确认交付分支与个人分支的差异仅为归档提交（步骤 29） | `git diff --stat dev-qingyang..personal-main`，改动应全在 `ai-collab/` 下 |

## 风险点与回滚

- **RK1 密钥顺序**：必须在首次启动前替换 `SYSTEM_AES_KEY`。若已用默认密钥启动并
  录入过模型凭据，之后更换密钥会导致这些凭据无法解密。回滚方式为清空数据卷重来，
  在最小验证阶段代价可接受，但正式使用后不可接受
- **RK2 Embedding 维度**：维度一旦与向量索引不匹配需重建索引并重新入库。
  务必在录入任何文档前确认为 1024 维
- **RK3 删除 `.temp-weknora`**：见阶段一第 4 步，需用户确认。可逆
- **RK4 误提交风险**：R1 完成前不要在 open-webui 仓库执行 `git add .`，
  `.temp-weknora/`（129MB）未被 `.gitignore` 排除
- **RK5 上游漂移**：新 clone 可能已超过 F2 记录的 `1e36e07`。若行为与 PRD 记录的
  事实不符，先核对 commit 再判断是否为上游变更所致

## `task.py start` 前的确认项

- [ ] 用户已批准 `prd.md`
- [ ] Docker Desktop 已装且 WSL2 后端可用
- [ ] 开发机可用内存满足 F10 估算（D6 已确认 16GB 及以上）
- [ ] 已明确阶段一第 4 步的删除操作需单独确认
