---
name: weknora-migration-context
description: "项目从 open-webui 转向 WeKnora 的动机、真实范围，以及\"引用不达标\"这一动机其实未被 WeKnora 完全解决的关键事实"
metadata: 
  node_type: memory
  type: project
  originSessionId: 2ce1c9a4-e950-4ac3-8854-8d118c6920ec
  modified: 2026-09-09T14:07:20.744Z
---

项目目标是为西门子 S120 变频器工业手册（参数章节约 2000 页）建 RAG 知识库问答系统。
2026-09-09 决定弃用 open-webui，改用 WeKnora（团队 fork：`OrangeSunrise/WeKnora`，
上游 `Tencent/WeKnora`，v0.8.0）。

**Why:** 用户认为 open-webui 对输出结果的文档引用做得不好。

**关键事实（与迁移动机相冲突，后续容易被遗忘）：** WeKnora 的引用只到分块级，
全链路没有页码，前端也没有任何 PDF 渲染器，点击引用只能跳到文档级页面，
不能定位到手册具体页、不能高亮。对 2000 页手册而言，迁移**并未真正解决**
最初的动机。用户已知情并决定本次不做页级引用改造。

**其他易被误判的事实：**
- WeKnora v0.8.0 是成熟产品而非脚手架（1984 个 Go 文件、876 个测试、
  仅 1 处非测试 TODO）。用户最初以为它"自带的都是开始示例"，实际只有配置层面
  需要补齐。不要沿用"这是个示例项目"的假设。
- 团队工作汇报里设计的混合检索（向量 + BM25 + RRF 融合）与 1024 维方案，
  WeKnora 已内建，无需自研或迁移。
- 团队汇报 §2 规划的自研栈（FastAPI + React + Milvus）已作废，不再是技术方向。

**How to apply:** 讨论检索质量或引用体验时，先确认页级引用是否已被重新纳入范围；
不要假设"换了 WeKnora 引用问题就解决了"。涉及 S120 参数表解析、参数编号元数据、
页级引用时，先查是否仍在 Out of Scope。

相关：[[framework-gaps-are-dropped]]
