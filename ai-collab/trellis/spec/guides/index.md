# Thinking Guides

> 本目录收录跨模块的通用规范与思考指南。撰写代码或文档前先查阅相关条目。

---

## Available Guides

| Guide | Purpose | When to Use |
|-------|---------|-------------|
| [文档撰写规范](./document-writing-style.md) | 交付文档的文风标准（正式、严谨、流畅，不口语化） | **撰写或修改任何面向交付的文档时（部署说明/使用手册/开发说明/README 等）** |
| [项目文档组织约定](./project-doc-organization.md) | document/ 目录与文档索引的组织方式 | **新建项目文档、整理文档结构、需要文档检索入口时** |
| [Vision MCP 调用规范](./vision-mcp-usage.md) | 图片输入自动调用 vision MCP，按意图决定是否注入工程上下文 | **用户输入包含图片时** |
| [图片生命周期管理规范](./image-lifecycle.md) | 会话内历史图片自动降级，防止 transcript 膨胀与 413 错误 | **会话内累积了多张截图或设计稿时** |

---

## When to Think About Document Writing Style（撰写文档文风）

- [ ] 正在撰写或修改面向交付的文档（部署说明、使用手册、开发说明、README 等）
- [ ] 文档将交付给企事业单位用户或客户
- [ ] 你写下了口语化、比喻性、调侃性或带括号补充说明的句子

→ 阅读并遵循 [文档撰写规范](./document-writing-style.md)。核心：正式严谨、行文流畅、
不口语化、不用解释性括号、保留全部技术信息。

## When to Think About Project Doc Organization（项目文档组织）

- [ ] 正在新建一份不属于某个具体模块的文档（如学习笔记、通用参考）
- [ ] 需要为项目建立或更新文档检索入口
- [ ] 文档散落各处、难以查找

→ 阅读并遵循 [项目文档组织约定](./project-doc-organization.md)。核心：根目录设 `document/`
存放无固定归属的文档，并维护 `document/文档索引.md` 作为全项目检索入口。

## When to Think About Vision MCP（图片输入处理）

- [ ] 用户输入中包含图片（截图、照片、设计稿等）
- [ ] 需要理解界面内容或提取文字

→ 阅读并遵循 [Vision MCP 调用规范](./vision-mcp-usage.md)。核心：自动调用 vision MCP，
按用户意图决定是否在 prompt 中注入工程上下文。
