---
name: weknora-agent-config-is-db-authoritative
description: WeKnora 内置智能体的配置与提示词首次启动即落库并实体化，之后改 config/prompt_templates 与 builtin_agents.yaml 一律无效
metadata: 
  node_type: memory
  type: project
  originSessionId: a19bb1d1-2899-458e-8d3c-d9a7ba0d08ea
  modified: 2026-09-10T04:21:09.412Z
---

WeKnora 首次启动时把内置智能体（如 `builtin-quick-answer`）写入 `custom_agents` 表，
并把 `system_prompt` / `context_template` 的**内容**实体化存进该行的 `config` JSON。
此后该行是唯一权威来源：改 `config/prompt_templates/*.yaml` 或 `config/builtin_agents.yaml`
再重启，对已落库的智能体**完全无效**（YAML 只回填 name / description / avatar）。

**Why:** 2026-09-10 排查「回答用英文且答非所问」时发现，团队此前为修图片幻觉而改的
`system_prompt.yaml` 从未生效——智能体里存的仍是旧版，仍要求「答案必须至少含一张图片」，
而语料是纯文本，模型只能编图片地址。同理 `embedding_top_k` 改 YAML 也不生效。
`document/2026-09-10_weknora-lite-原生Windows部署说明.md` §12.3 声称「改模板后重启即生效」，
对智能体路径是错的。

**How to apply:**
- 要改内置智能体的提示词、模板选择或检索参数，走 `PUT /api/v1/agents/{id}`
  （需系统管理员）或 GUI 的「智能体」页，改完立即生效、无需重启。
- 改 YAML 只对**全新部署**有意义；两处都改可以保证新环境正确。
- 判断某项配置是否已落库：`select config from custom_agents where id='builtin-quick-answer'`。
- 同理，会话把当时的智能体配置快照进 `sessions.agent_config`，改配置后必须**新建会话**才生效。

相关：[[weknora-migration-context]]、[[framework-gaps-are-dropped]]
