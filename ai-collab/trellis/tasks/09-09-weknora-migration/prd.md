# WeKnora 框架迁移与功能完善

## Goal

以 WeKnora v0.8.0 替换 open-webui 作为 S120 知识库问答系统的底座，把框架从开箱状态
配置成一个能跑通的最小可用系统：标准 Docker Compose 部署、本地 Ollama 模型接入、
修正上游不安全的默认密钥，并沉淀可复现的部署文档。

本任务是**配置与部署**，不含功能开发。WeKnora 未提供的能力一律不补（见 D0）。

## Background

### 项目最终目标（来自工作汇报）

为西门子 S120 变频器工业手册构建 RAG 知识库问答系统。

- 参考文档：`document/知识库搭建工作汇报-2026年8月7日/知识库搭建工作汇报-2026年8月7日.md`
  （用户说明：部分内容已过时，仅用于理解开发目的）
- 核心数据对象：S120 参数章节约 1900~2000 页，参数编号（如 p1082）、名称、数据类型、
  取值范围、默认值、说明、枚举值、数据组归属（汇报 §6）
- 团队：张清扬、刘文楚、常建辉。项目文档站 https://orangesunrise.github.io/RAG-SGAI-Public/
- 示例语料：中文维基百科构建的工业示例知识库，约 1 万文档 / 4~6 万 chunks / 1024 维
  （汇报 §5），用于真实西门子数据就位前并行开发
- 汇报 §7 的本地化约束：消费级笔记本、Intel 处理器、1.7b~4b 参数级量化 LLM。
  该节选定的 OpenVINO + Qwen3-4B-int4-ov 路线在本项目中已搁置（见 D5），
  但"本地小模型 + 消费级硬件"这一约束继续成立
- 已有进展（汇报 §8）：Milvus Collection Schema、语义检索模块代码框架、
  混合检索方案设计（向量 + BM25，RRF 融合）

### 汇报原技术选型 vs WeKnora 实际栈

汇报 §2 规划的是自研栈，与 WeKnora 差异显著。但由于 open-webui 侧没有任何自研业务
代码（F1）、汇报 §8 的检索设计又已被框架内建（F9），**这些差异不产生实际迁移成本**，
只意味着原技术选型作废：

| 层次 | 汇报原计划 | WeKnora 实际 |
| --- | --- | --- |
| 后端 | Python 3.12 + FastAPI | Go |
| 前端 | React 18 + TS + Ant Design | Vue 3 + TS + Vite |
| 向量库 | Milvus | 10 种驱动（pgvector / Milvus / Qdrant / Weaviate / OpenSearch / Doris / SQLite 等） |
| LLM 编排 | LangChain / LlamaIndex | 自研 Go 编排 + ReAct Agent |
| 文档解析 | Docling + Label Studio + Python | docreader（Python + gRPC，约 17 个解析器） |

## Confirmed Facts

### F1. open-webui 侧没有任何自研业务代码，迁移沉没成本≈0

`dev-jianhui` 相对 `main` 共 9 个提交、186 文件、+26171 行，全部为：文档（`document/`、
`.trellis/`）、Trellis 工具链（`.agents/`、`.claude/`）、删除无用静态资源
（`backend/open_webui/static/*`）、`pyproject.toml` 增 5 行。

**没有任何 RAG / 检索 / 引用相关的自研功能代码。** 本次不是"移植已有功能"，而是
"改换技术底座、重新开始功能开发"。

### F2. WeKnora 检出物是 Tencent 上游 v0.8.0 的浅克隆，尚未做任何本地改动

- 位置 `.temp-weknora/`（当前 untracked）；remote 为团队 fork
  `https://github.com/OrangeSunrise/WeKnora.git`，上游 `github.com/Tencent/WeKnora`
  （README_CN.md:30）
- `git rev-list --count HEAD` = 1，存在 `.git/shallow` → depth=1 浅克隆
- HEAD `1e36e07 feat(mcp): add per-tool enable toggle (#2989)`，`VERSION` = `0.8.0`

### F3. WeKnora v0.8.0 是成熟产品，不是脚手架

- 1984 个 Go 文件、876 个测试文件、12 条 CI workflow；`frontend/src` 190 个 Vue 组件；
  `docreader` 61 个 Python 文件
- **全部 1984 个 Go 文件中仅 1 处非测试 TODO**；此前统计到的 76 处
  TODO/FIXME/not-implemented 几乎全在 `*_test.go`，属测试替身（fake/mock），非功能缺口
- 自带 VitePress 文档站 `website-docs/`（六大板块）、`docs/` 约 40 篇专题文档、
  `docs/swagger.json`；自述覆盖约 360 个 API 端点、约 150 个环境变量（README_CN.md:63）
- 扩展点均自带多个可用驱动，而非"接口 + 一个样例"：10 种向量库、12 种 Embedding、
  9 种 Rerank、10 种网络搜索、6 种数据源、约 17 种文档解析器、多种分块策略
- 真正的示例/模板：`examples/mcp-demo`、`examples/skills/pdf-processing`、
  `rerank_server_demo.py`、`dataset/`、`docs/poc`、`testdata`

### F4. 配置层面确实需要开发者补齐（对应用户 Q1 的判断）

- `.env.example` 共 730 行，仅两处标注 `⚠️ 必填`：B1 数据库（docker 下有可用默认值）、
  F1 密钥
- **F1 密钥是真实的生产陷阱**：`JWT_SECRET=weknora-jwt-secret`、
  `SYSTEM_AES_KEY=weknora-system-aes-key-32bytes!!` 是可直接启动的不安全默认值；
  且 `SYSTEM_AES_KEY` 一旦丢失，已加密的凭据不可恢复
- 开箱状态**没有配置任何模型**，模型通常经 UI 配置 → 首次启动的观感确实像"空骨架"
- `config/builtin_models.yaml.example`：可选的声明式模型预置
- 默认关闭、需显式开启：Docker 沙箱、Neo4j 知识图谱、OIDC、Langfuse

### F5. lite profile 不可用作正式系统（推翻先前判断）⚠️

`.env.lite.example`（57 行）提供零依赖 profile：SQLite + FTS5/sqlite-vec + 本地文件存储
+ 内存流 + Ollama，表面上契合消费级笔记本约束。但 `docs/LITE.md` 明确限制：

- **只带 "Simple" 解析引擎** —— 对 2000 页 S120 PDF 不够用，这是硬伤
- 单空间、无共享空间、无注册 —— 三人团队失去协作能力

lite 的混合检索本身是可用的：`internal/application/repository/retriever/sqlite/repository.go`
按维度建 `vec0` 虚拟表（`:114`，`vec0(embedding float[%d] distance_metric=cosine)`，
由 `:40` 的 `vecTables map[int]bool` 跟踪），并配 contentless FTS5 表 +
**手写 bigram 分词**（`:73`、`initFTS5` at `:62`；bigram 正是为了绕开 FTS5 不能分中文词）。

但 `vec0` **没有 ANN 索引，是暴力扫描**，4~6 万 chunks × 1024 维会明显慢于 HNSW。

### F6. WeKnora 引用管线本身是生产级实现

- 采用 handle 协议（`cN`/`wN`/`dN`/`bN`）在 LLM 与前端间传递引用；对模型编造的
  handle **fail-closed**（丢弃而非渲染），这点比多数实现严格
- 前端已有 `ChatCitationFloat.vue`（引用浮层）、`ChatReferencesDrawer.vue`（引用抽屉）、
  `useChatCitationPopover.ts`；分块弹层支持版本感知坐标
- `Chunk.Metadata`（JSON）可承载自定义字段，`ContextHeader` 已捕获标题面包屑路径

### F7. 但引用只到"分块级"，全链路没有页码，也没有原文跳转高亮 ⚠️

这是与本次迁移**初始动机直接冲突**的发现：

- **无页码**：Go 侧 `page_num` 零命中。docreader 的 proto 只返回扁平
  `markdown_content`；`docreader/.../docx_parser.py:105` 内部**确实跟踪了 page_num**，
  但在 proto 边界被抹平
- **前端没有任何 PDF 渲染器**。文档预览是文档级的，不支持滚动定位到引用位置，
  也没有高亮
- 引用的跳转图标指向 `/platform/knowledge-bases/{kbId}?knowledge_id={id}`，
  仅文档级导航
- **对参数表不友好**：分块基于扁平 markdown 字符串，参数行跨块切分会丢失表头行，
  导致引用片段自身不可核验
- **没有结构化字段检索**：无法按"参数编号 = p1082"做元数据过滤查询
- `Chunk.Metadata` / `ContextHeader` 虽可承载 S120 字段，但**当前没有任何代码从参数表
  填充它们**

对 2000 页手册而言，"引用到某个 chunk"与"跳到手册第 1432 页并高亮"是两回事。
换到 WeKnora 后引用体验会明显好于 open-webui（有浮层、抽屉、fail-closed 校验），
但仍达不到页级可核验 —— 即本次迁移的初始动机并未被完全解决，用户已知情（D2）。

### F8. Windows 只能走 Docker Desktop + WSL2，没有原生路径

- 全仓库没有 `.bat` / `.ps1`；`Makefile` 与 `scripts/` 全为 POSIX shell
- Go 侧需要 cgo，原生 Windows 构建不成立
- 结论：Docker Desktop（WSL2 后端）是唯一支持路径，需计入 WSL2 自身的内存开销

### F9. 团队汇报 §8 设计的混合检索，框架已经实现，无需自研

`internal/application/service/knowledgebase_search_fusion.go:84` 的 `fuseWithRRF`
即 RRF 融合。向量 + 关键词混合检索为框架内建能力。

汇报 §8 "混合检索方案设计（向量 + BM25，RRF 融合）"在 WeKnora 下**是既有功能**，
不是待开发项。同理汇报 §8 的 Milvus Collection Schema 与语义检索代码框架亦无需迁移。

### F10. 标准 Compose 默认栈用 ParadeDB，原生支持 BM25 + 向量

默认 `docker-compose.yml` 起 5 个服务：frontend、app、docreader、postgres
（`paradedb/paradedb:v0.22.2-pg17`，自带 BM25 + pgvector）、redis。

内存估算：容器约 2~3.5GB + WSL2 约 1~2GB + 4B 量化模型约 3~4GB。
**16GB 内存可行，8GB 不可行。**

### F11. 1024 维是一等公民

migration `000059` 直接面向 bge-m3 1024 维建 HNSW（halfvec）索引，
与汇报 §5 的 1024 维规划完全对齐，无需改动。

### F12. WeKnora 零 OpenVINO 支持；本地模型的接入点是 generic provider ⚠️

- 代码中**没有任何 OpenVINO 集成**
- 本地 LLM 的现成路径只有两条：Ollama（原生支持），或 `generic` provider
  （OpenAI 兼容 + 自定义 base URL）
- 本地 Embedding 同样只有 ollama 与"OpenAI 协议 + 自定义 base URL"两条
- Rerank **没有本地 provider**，但 `OpenAIReranker` 的 baseURL 可配；
  仓库根的 `rerank_server_demo.py` 正是自建 rerank 服务的模板
  （但它用 torch/CUDA，需改造为 CPU 或 OpenVINO）

汇报 §7 已实测并选定 OpenVINO + Qwen3-4B-int4-ov（Intel iGPU）。要在 WeKnora 中沿用
这一投入，需自写一层 OpenAI 兼容 HTTP shim 包住 `openvino_genai.LLMPipeline`。
按 D0/D5 该 shim 不做，OpenVINO 路线搁置，改用 Ollama。

### F13. Docker 路线在本机网络环境下不可行；Lite 单二进制路线可行 ⚠️

**Docker 侧阻断（实测）：**

- `docker compose pull` 无法拉取镜像：
  `Get "https://registry-1.docker.io/v2/": reading HTTP CONNECT: unexpected EOF`
- 拉取失败后 `up -d` 回退到本地构建 docreader，构建时 `apt-get` 撞 502
  （来自 `198.18.0.56`，Clash 的 fake-ip 段）
- 宿主机本身可达 Docker Hub（`curl https://registry-1.docker.io/v2/` 返回 401），
  系统代理为 `127.0.0.1:7890` 且 Clash 监听 `0.0.0.0:7890`（Allow LAN 已开）
- 已尝试将 Docker Desktop 代理改为 manual 指向宿主机（先
  `host.docker.internal:7890`、后 IPv4 `192.168.1.2:7890`），
  经完整 stop→确认关闭→start 循环后仍报同样的 CONNECT EOF
- 副产物：`host.docker.internal` 在容器内解析为 IPv6 `fdc4:f303:9324::254`
- Docker Desktop 配置已备份至
  `%APPDATA%\Docker\settings-store.json.bak-before-proxy`，可回滚

**Lite 路线可行（实测）：**

- `Makefile:263-286` 提供 `build-lite` / `run-lite` / `package-lite`：
  单 Go 二进制，`-tags "sqlite_fts5"`，SQLite + 内存队列，前端打进 `web/`，
  `SKIP_FRONTEND=1` 可跳过前端构建
- `docs/LITE.md` 定位为"单应用、零依赖"，不依赖独立数据库与消息队列
- 不需要 docreader / Postgres / Redis 容器；Lite 跑在宿主侧，
  Ollama 直接走 `localhost:11434`，无需 `host.docker.internal`
- ~~WSL2 Ubuntu 24.04.4 LTS 已就绪，自带 gcc 13.3.0（满足 cgo）、GNU Make 4.3、
  node v22.22.2；仅缺 `go.mod` 要求的 Go 1.26.0~~
  **已作废（D9）**：改走原生 Windows，WSL2 不再是运行位置。
  且该 WSL 实例其后不可用（`WslService/HCS_E_CONNECTION_TIMEOUT`，
  疑为反复 stop/start Docker Desktop 的连带影响），未修复也不再需要
- v0.8.0 的 GitHub Release **无二进制附件**，必须自行构建
- 关键优势：Go 模块可走 `goproxy.cn`，为国内源，
  绕开了阻断 Docker 的那条代理链路（原生 Windows 同样受益）

### F14. 原生 Windows 构建与运行已实测通过 ✅

- 产物 `WeKnora-lite.exe`（strip 后 217MB），注册 429 条 gin 路由，
  `/health` 返回 200，SQLite + sqlite-vec + FTS5 bigram 检索就绪
- 工具链走 scoop 用户级安装：Go + mingw-w64 gcc 16.1.0（posix-seh）
- 三个构建阻塞及解法见 D9
- 前端需用 Node 22 单独构建后拷到 `web/`；`fnm exec --using=... npm ci`
  在 Windows 上会报 "Can't spawn program"（npm 是 `.cmd` shim），
  须直接用 `node.exe` 调绝对路径的 `npm-cli.js`
- `.env.lite.example` 自带 `OLLAMA_BASE_URL=http://127.0.0.1:11434`
  但未放行该地址，SSRF 校验会拦回环地址导致初始化报
  "hostname 127.0.0.1 is restricted"。属上游配置缺口，
  须补 `SSRF_WHITELIST_EXTRA=127.0.0.1`
- `/initialization/initialize` 建出的模型 `tenant_id=0`，
  而 `modelRepository.GetByID` 按 `(tenant_id = ? OR is_builtin = true)` 过滤，
  导致模型对租户不可见（`GET /models` 空、`GET /models/{id}` 404）。
  根因：`initialization.go` 的 `toModel()` 与 `modelService.CreateModel` 均未设
  TenantID。按 D0 不改 Go 代码，改用 `POST /api/v1/models`（该 handler 从上下文取
  TenantID）建模型，再用 `PUT /initialization/config/{kbId}` 绑定

### F15. 示例知识库 `wiki-kb-results` 的性质 ✅

- 内容是**中文维基工业类条目**，不是 PDF：`data/filtered/industrial_wiki_clean.jsonl`
  9,717 篇 → 24,772 块。**因此 PDF 解析阻塞在本阶段作废**，Simple 解析器够用
  （`builtin_converter.go:14-43` 的 `simpleFormats` 只含 md/txt/csv/json，无 pdf）
- 向量为 BGE-M3 / 1024 维 / Cosine（`qdrant/.../config.json`），与现有部署天然对齐
- 归档里的 613MB 预计算向量与 Qdrant 集合**不作为入库源**：绕过 WeKnora 的
  knowledge/chunk 记录会废掉引用能力，而引用正是换框架的初衷。
  实测 Ollama 嵌入吞吐约 10 chunks/s，全量重嵌约 41 分钟，成本可接受
- 真正价值是**评测基线**：`reports/retrieval_report_v2.md` 给出 Recall@1 64%、
  @5 78%、@10 88%、MRR 0.7157、NDCG@5 0.9732，平均延迟 38.9ms；
  外加 `07_evaluate_v2.py` 里 50 道带 `expected_concepts` 的题
- `title` 字段存的是正文首句而非标题，需自行抽取
- 50 道题共涉及 224 个概念，其中 **26 个在全量语料里完全不存在**
  （Modbus、PROFIBUS、PROFINET、伺服、异步电机、矢量控制、磁场定向、现场总线、
  工业以太网、扫描周期、Ziegler-Nichols 等）。这解释了基线里
  **应用类 Recall@5 仅 30%** —— 是语料覆盖缺口，不是检索器缺陷。
  按概念贪心 set-cover 选 300 篇可覆盖 198/224，即语料中存在的概念全覆盖，已达上限

### F16. qwen3:4b 会污染文档摘要，并经索引形成提示注入 ⚠️

链路：入库时生成文档摘要 → 摘要被存为 `ChunkTypeSummary` 且 `IsEnabled=true`
（`knowledge_process.go:1310`）→ 问答时被检索进上下文 → LLM 把其中的指令样式文本
当成自己的任务，无视用户提问、改去总结文档。

- **根因是模型能力，不是框架 bug**：`knowledge_process.go:932` 已正确传
  `Thinking: &false`，`ollama.go:109` 也确实下发 `think: false`。
  直接调 Ollama 复现：qwen3:4b 在 `think:false` 下返回
  `thinking` 字段为空、`content` 1649 字**全是推敲过程**——
  模型不遵守"直接输出总结正文，不要前缀"，框架无从剥离
  （`StripThinkBlocks` 只能处理 `<think>` 标签，此处没有标签）
- 加重因素：`ollama.go:145-148` 的兜底 `if responseContent == "" && Thinking != ""`
  会把思维链当正文；`validateSummaryOutput` 只校验非空
- 实测后果：两次不同提问都返回同一段无关的雷达摘要，而检索本身完全正确
  （命中的正是含答案的原文块），说明**检索层没问题，是生成层被注入劫持**
- 摘要 chunk **不出现在 `GET /chunks/{knowledgeID}` 列表里**（列表只返回 text 类型）
  却参与检索，排查时容易漏掉
- 摘要 chunk 生成是无条件的（只要摘要非空且 KB 需要向量化），config.yaml 无开关，
  故配置层唯一解法是换一个指令遵循更好的模型（见 D10）
- 换 `qwen2.5:7b-instruct` 后 reparse 两篇文档，摘要恢复正常（777/705 字，纯事实），
  同一提问的答案随即正确

### F17. 最小端到端验证通过，引用能力符合换框架的初衷 ✅

以 2 篇文档、`chunk_size=512 / overlap=64` 实测全链路：

- 上传 → 解析 → 分块 → 嵌入 → 混合检索 → 带引用生成，各环节均通
- 检索层始终正确：问 AC4000 时含答案的原文块稳定排第一
  （即使生成层曾被 F16 的注入劫持，检索排序也没错）
- 生成层给出**行内引用标记**：
  `<kb doc="AC4000型电力机车.md" chunk_id="428ab43b-..." kb_id="c061620e-..." />`
  即 chunk 级、锚定到具体分块，加上 SSE 的 `references` 结构化数组。
  这正是 open-webui 做得差、促成本次换框架的能力
- 负样本（问知识库里没有的 S120 P1120 参数）正确拒答，
  既未编造内容也未伪造引用标记
- SSE 事件类型是分开的：`agent_query` / `tool_call` / `tool_result` /
  `references` / `thinking` / `answer` / `complete`。
  客户端**必须按 `response_type` 分流**，把 `thinking` 和 `answer` 的
  `content` 混拼会得到看似跑偏的答案（本次排查中先踩了这个坑）
- 残留瑕疵：qwen2.5:7b-instruct 偶尔在答案里编造图片链接
  （如 `![...](https://example.com/....jpg)`），属模型瑕疵，不影响引用正确性
### F18. Windows Git Bash 不能用 argv 传中文给 curl ⚠️

同一个根因造成了两处故障，凡是经 Git Bash 命令行传中文都会中招：

- **上传失败**：中文文件名按 ANSI 代码页进 argv，curl 发出的 multipart filename
  不是合法 UTF-8，被 `ValidateInput`（`internal/utils/security.go:80`）
  拒为"文件名包含非法字符"
- **数据被写坏**：`curl -d '{"name":"工业维护知识库"}'` 内联中文同样按 GBK 送出，
  知识库名称与描述在库里存成了非法 UTF-8，界面显示为 `��ҵά��֪ʶ��`
- **不可逆**：落库时非法字节已被替换为 U+FFFD。只有那些 GBK 字节对**恰好构成
  合法 UTF-8 序列**的字侥幸留存（如 业=D2B5、维=CEAC、知=D6AA、识=CAB6），
  可反解出 `？业维？知识？`，据此确认原值为「工业维护知识库」；描述残缺过多已放弃还原
- **规避**：所有含中文的请求一律用 Python requests（显式 UTF-8），
  或先用 Write 工具落成 JSON 文件再 `curl -d @file`。
  本次模型创建走的正是后者，因此未被污染——已核对 models / knowledge-bases /
  sessions 三类资源，除知识库外无其他乱码

### F19. 答案里的破图源自默认提示词强制放图，与纯文本语料冲突 ✅已修

界面上出现加载失败的图片，既不是从资料摘取也不是生图，而是模型编造了
`https://example.com/apg68_system_composition.png` 这类 URL，前端把模型输出当
Markdown 渲染，遇到 `![alt](url)` 就照发 `<img>` 请求，于是显示为破图占位。

- **根因不在模型**：`config/prompt_templates/system_prompt.yaml` 的 `default_kb`
  模板原文要求 "the final answer **MUST** include at least one relevant image…"
  并 "silently verify that the answer satisfies this image requirement"，
  同时另一条又写 "must not be fabricated"。检索结果里没有任何图片时两条硬性
  要求无解冲突，7B 模型的解法是造一个 URL 交差
- **本知识库不可能有图**：`vlm_config.enabled=false`、
  `image_processing_config.model_id=""`，加上 Simple parser + 纯文本 wiki 语料，
  没有任何环节会把图片写进 chunk。WeKnora 默认场景是 PDF/docx 走 VLM 解析、
  chunk 自带图片 URL，那种前提下强制放图是合理的——属场景不匹配，非框架缺陷
- **修法（纯配置层）**：把强制放图改为条件放图，并显式禁止编造 URL。
  模板由 `internal/config/config.go:1080` 在启动时从磁盘目录读取，
  **不是 `go:embed`**，因此只需改 yaml + 重启，无需重新编译、不碰 Go 代码
- **验证**：重启后复问两个原本编图的问题（apg68 目标数、AC4000 研制单位），
  answer 中 `![...]()` 与 `http(s)://` 均为零
- **残留**：qwen2.5:7b-instruct 仍会做超出原文的推断（如 AC4000 那题声称
  "文档中并未提及研制时间"并推测为 20 世纪末，而检索块内明确写了 1996 年），
  属模型能力问题，不在本轮配置层范围

## Decisions

### D0 统领原则：框架没有的功能就作废，先做最小实现

用户明确要求（Q4）："weknora 里没有的就作废"、"先做最小实现"。

本任务**不为 WeKnora 缺失的能力写任何自研代码**。凡是框架不具备的（页级引用、
OpenVINO 接入、本地 rerank、S120 参数表结构化抽取），本次一律不做。
初期也**不以 2000 页 S120 手册作为开发数据**，用极小样本跑通端到端即可。

### 其余决策

- **D1（Q1）** 工作范围＝配置与部署层面，非功能重写
- **D2（Q2）** 页级引用与原文跳转高亮**不纳入**（跨 docreader / 分块 / 检索 / 前端
  四层改造）。先跑通部署，实测分块级引用的实际观感后再议
- **D3（Q3）** WeKnora fork **全深度 clone 到独立仓库** `D:\D_Code\WeKnora` 开发；
  open-webui 仓库降为文档/归档；移除 `.temp-weknora/`
- **~~D4（F5）~~ 已推翻** 原决策为"走标准 Docker Compose、不用 lite"。见 D8
- **~~D8~~ 已推翻（运行位置部分）** 原决策为"Lite 单二进制在 **WSL2 Ubuntu** 内构建运行"。
  「走 Lite 单二进制、不用 Docker」这一半仍然有效并由 D9 继承；
  「在 WSL2 内运行」这一半被 D9 推翻 —— 用户明确要求
  "weknora 不应该装在 docker 里而是应该装在 windows 里"，
  而 WSL2 是 Linux 虚拟机，不满足"装在 Windows 里"。
  推翻 D4 的两条理由（D9 继续沿用）：
  1. **D4 的论据与 D0 自相矛盾** —— D4 排除 lite 的理由是"Simple 解析引擎撑不住
     2000 页 S120 PDF"，但 D0 已明确初期不以 2000 页手册为开发数据。
     用户直接指出了这一点（"应该是不需要用到 docker 的，你的路线有问题"）
  2. **Docker 路线被网络环境阻断** —— 见 F13
  Lite 的代价（单空间、无共享空间、仅 Simple 解析）在最小实现阶段可接受；
  真正需要多空间协作或高精度解析时再评估切标准版
- **D5（Q4）** 本地模型走 **Ollama**（框架原生支持，零自研）。
  按 D0 作废：不写 OpenVINO shim —— 汇报 §7 的 OpenVINO/Qwen3-4B-int4-ov 投入
  在本项目中搁置；**本次不启用 rerank**（F12：无本地 provider）
- **D6（Q4）** 开发机内存 16GB 及以上，F10 的资源估算成立，标准 Compose 可行
- **D9（推翻 D8 运行位置部分）** Lite 单二进制在**原生 Windows** 上构建并运行，
  不用 Docker、也不用 WSL2。用户明确确认"确定要原生 Windows"。
  继承 D8 的「走 Lite、不用 Docker」，并沿用 D8 记录的推翻 D4 的两条理由。
  已验证可行（见 F14），构建三个真实阻塞均已解决：
  1. `sqlite-vec` 以 `-DSQLITE_CORE` 编译，需要系统 include 路径提供 `sqlite3.h`，
     而该模块只带 `sqlite-vec.c/.h`。解法：从 `mattn/go-sqlite3` 的 amalgamation
     取头文件暂存到 `.build/include`，版本与实际链接的 SQLite 3.46.1 精确一致
  2. DuckDB 预编译静态库要求 emulated-TLS 的 libstdc++，与 mingw 16.1.0 的
     native TLS ABI 不匹配（`__emutls_v._ZSt11__once_call` 未定义）。
     DuckDB 被无条件 import 无法裁掉，解法：`-tags duckdb_use_lib` + 官方
     DuckDB v1.5.2 Windows DLL，`duckdb.lib` 复制为 `libduckdb.dll.a` 作 mingw 导入库
  3. gojieba 在 `internal/types.init()` 阶段崩溃（`Exception 0xc0000005`），
     原因是 PATH 上存在不匹配的 `libstdc++-6.dll`。解法：
     `-static-libstdc++ -static-libgcc`（不用完整 `-static`，否则 duckdb 导入库失效）
  另需 `EDITION=lite` 的 ldflag，否则 `handler.Edition != "lite"`，
  `serveFrontendStatic` 不挂载，前端请求会落到 API 路由返回 401
- **D10（F16）** 生成模型由 `qwen3:4b` 换为 **`qwen2.5:7b-instruct`**（Ollama，4.7GB）。
  理由：qwen3:4b 不遵守"直接输出总结正文"，把推敲过程写进摘要正文，
  经索引后劫持问答生成（F16）。qwen2.5:7b-instruct 无 thinking 行为、
  同一提示下输出 244 字干净摘要。属配置层调整，未改任何 Go 代码，符合 D0/D1。
  嵌入模型仍为 `bge-m3`（1024 维，与 F15 的示例库对齐）
- **D7** 本 Trellis 任务**一次做完 R1~R6**，不拆父子任务；`D:\D_Code\WeKnora`
  不建 Trellis 配置。部署文档落在 open-webui 的 `document/` 并登记到文档索引，
  与 D3 中 open-webui 降为文档/归档仓的定位一致。
  依据：WeKnora 的 `.env` 被 `.gitignore:2` 的 `.*` 规则排除，R2~R5 在 WeKnora 仓内
  几乎不产生可提交产物，唯一持久交付物是 R6 文档；WeKnora 仓保持纯净上游 fork
  也最利于后续同步上游

## Requirements

- **R1 仓库归置**（D3）全深度 clone WeKnora fork 到 `D:\D_Code\WeKnora`，
  并配置 upstream 指向 `Tencent/WeKnora`；移除 open-webui 内的 `.temp-weknora/`
  （129MB 未跟踪且未被 `.gitignore` 排除，有误提交风险）
- **R2 部署跑通**（D4/F8）Docker Desktop + WSL2 下以标准 `docker-compose.yml`
  启动 5 服务栈（frontend、app、docreader、paradedb、redis），Web UI 可访问
- **R3 模型接入**（D5）Ollama 提供 LLM 与 Embedding；Embedding 选 1024 维模型
  以对齐 migration `000059` 的 HNSW/halfvec 索引（F11）。不启用 rerank
- **R4 密钥加固**（F4）替换 `JWT_SECRET`、`SYSTEM_AES_KEY` 的不安全默认值，
  并建立 `SYSTEM_AES_KEY` 保管与备份约定 —— 丢失即无法恢复已加密凭据
- **R5 最小验证**（D0）用极小样本（数篇文档量级，非 2000 页手册、非万级维基语料）
  跑通端到端：上传 → 解析 → 检索 → 带引用回答，并确认引用浮层/抽屉可用
- **R6 文档沉淀**（D7）部署与配置步骤成文，落在 open-webui 的 `document/` 并登记到
  文档索引，供团队另两名成员（张清扬、刘文楚）复现

## Acceptance Criteria

- [ ] **AC1**（R1）`D:\D_Code\WeKnora` 为全深度 clone，`git rev-list --count HEAD` > 1
      且无 `.git/shallow`；`git fetch upstream` 可拉取 Tencent 上游
- [ ] **AC2**（R1）open-webui 仓库内 `.temp-weknora/` 已移除，不再出现在 `git status`
      的未跟踪列表中
- [ ] **AC3**（R2）`docker compose ps` 显示 frontend / app / docreader / postgres /
      redis 五服务健康；浏览器可打开 Web UI 并完成注册登录
- [ ] **AC4**（R3）设置页中 LLM 与 Embedding 均指向本地 Ollama 且连通性测试通过；
      Embedding 维度为 1024
- [ ] **AC5**（R4）`.env` 中 `JWT_SECRET` 与 `SYSTEM_AES_KEY` 均非上游默认值；
      `SYSTEM_AES_KEY` 已按约定备份并记录位置
- [ ] **AC6**（R5）上传至少 1 篇样本文档，解析状态为完成，分块数 > 0
- [ ] **AC7**（R5）就该文档提问后，回答中出现引用标记，点击可打开引用浮层/抽屉
      并显示对应分块原文
- [ ] **AC8**（R6）部署文档存在且经第二人按文档从零复现成功（或至少可在干净环境
      按文档走通 AC3~AC7）

## Out of Scope

- 页级引用与原文跳转高亮（D2）
- S120 参数章节结构化抽取（汇报 §6 的 Docling + Label Studio + Python 流水线）
- 参数表分块修复与参数编号元数据入库（F7 已识别的缺口，但属功能开发，非配置部署）
- 混合检索与 RRF 融合的自研实现 —— 框架已内建（F9）
- lite / SQLite profile（D4）
- open-webui 侧任何代码改动 —— 该仓库无自研业务代码（F1）
- OpenVINO 接入 shim（D5）—— 汇报 §7 的 OpenVINO 投入在本项目中搁置
- rerank 启用与本地 rerank 服务（D5/F12）
- 万级维基示例语料入库（D0）—— 初期只用极小样本
- 汇报 §2 原自研栈（FastAPI / React / Milvus）与 §8 已写的语义检索代码 —— 不迁移（F9）

## Open Questions

无阻塞项。Q1~Q4 均已收敛为 D0~D7。

具体 Ollama 模型选型（LLM 与 1024 维 Embedding）在实施时按机器实际表现确定，
不阻塞规划。

## Notes

**不写 `design.md`。** 本任务按 D0/D1 为纯配置与部署，零自研代码，框架原样使用，
没有需要设计的架构、数据流或契约。技术决策已全部收敛在 Decisions 与 Confirmed Facts 中。
执行顺序、验证命令与回滚点写入 `implement.md`。

**风险提示：** `.temp-weknora/` 是 129MB 未跟踪目录且未被 `.gitignore` 排除
（第 281 行的 `.temp` 不匹配 `.temp-weknora`），在 R1 完成前应避免在 open-webui 仓库
执行 `git add .`。



