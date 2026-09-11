# AI 协作记录

本目录归档 WeKnora 框架迁移与原生 Windows 部署工作的完整 AI 协作过程，包含
Trellis 任务产物与 Claude Code 会话记录。归档目的是让接手者不仅看到交付结果，
还能追溯每个技术决策的由来与被否决的方案。

本目录仅存在于个人存档仓库 LightlyFloat/RAG-SGAI-WeKnora 的 `main` 分支。
团队交付分支 OrangeSunrise/WeKnora 的 `dev-qingyang` 不含本目录，两个分支的
全部差异即为本目录。内容边界的划分依据见 `document/仓库与分支说明.md`。

## 一、trellis/

Trellis 是本次工作使用的任务管理与规范约束框架，原始位置为
`D:/D_Code/open-webui/.trellis/`。

| 路径 | 内容 |
| --- | --- |
| `workflow.md` | 三阶段开发流程的完整定义，依次为规划、执行、收尾 |
| `config.yaml` | 框架配置 |
| `spec/guides/` | 项目规范。`document-writing-style.md` 约束了 `document/` 下全部交付文档的行文标准，`project-doc-organization.md` 约束了文档索引的维护方式 |
| `tasks/09-03-onboard-openwebui/` | 前一阶段任务，open-webui 部署上手 |
| `tasks/09-09-weknora-migration/` | 本次任务，WeKnora 框架迁移与功能完善 |

每个任务目录含五个文件：`prd.md` 记录需求、已确认事实与决策，`implement.md`
记录执行计划、验证命令与风险点，`implement.jsonl` 与 `check.jsonl` 为子代理的
上下文清单，`task.json` 为任务状态。

要理解本次工作的取舍，从 `tasks/09-09-weknora-migration/prd.md` 读起，其中的
决策 D0 是贯穿全程的红线：不为 WeKnora 缺失的能力编写任何自研代码。

## 二、claude/

Claude Code 会话的原始导出，原始位置为
`C:/Users/<用户名>/.claude/projects/D--D-Code-open-webui/`。文件格式为
JSON Lines，每行一条消息或一次工具调用。

| 文件 | 体积 | 对应工作 |
| --- | --- | --- |
| `1f924ce4-….jsonl` | 47KB | 最早的一次会话 |
| `1577963a-….jsonl` | 2.9MB | 迁移调研与部署推进 |
| `a19bb1d1-….jsonl` | 4.9MB | 部署推进与配置调优 |
| `2ce1c9a4-….jsonl` | 8.1MB | 浏览器功能测试，对应 `document/测试记录.md` |
| `7391808e-….jsonl` | 见文件 | 交付发布与文档补齐，即产出本目录的会话 |

`<会话 ID>/subagents/` 存放该会话派生的子代理记录，同名 `.meta.json` 标明子代理
的类型与用途。已归档的子代理共五项：本地部署可行性调研、引用能力深度调研、
框架完成度审计、交付文档完整度评估、交付文档补齐。

`memory/` 是跨会话记忆，`MEMORY.md` 为索引，其余三个文件各记录一条需要跨会话
保持的结论，包括框架缺口的处理原则、迁移动机，以及智能体配置以数据库为准这一
容易踩错的事实。

## 三、关于原始会话记录

会话记录为原始导出，未做任何删减或脱敏，因此其中包含开发过程中读取过的配置
文件内容与命令输出。任何新部署都必须按 `document/操作说明.md` 自行生成
`SYSTEM_AES_KEY`、`TENANT_AES_KEY` 与 `JWT_SECRET` 三项密钥，不得沿用记录中
出现的任何值。

会话记录中的绝对路径为原开发机路径，包括 `D:/D_Code/WeKnora` 与
`D:/D_Code/open-webui`，在其他环境中不可直接照搬。

## 四、阅读建议

按以下顺序可用最短路径掌握本次工作的全貌：

1. `document/工作说明.md` —— 做了什么、为什么迁移、交付边界。
2. `trellis/tasks/09-09-weknora-migration/prd.md` —— 决策依据与已确认事实。
3. `document/问题与解决.md` —— 踩过的坑与解决方法。
4. `document/操作说明.md` —— 从零复现部署与知识库导入。
5. 需要追溯某个具体结论时，再检索对应的会话记录文件。
