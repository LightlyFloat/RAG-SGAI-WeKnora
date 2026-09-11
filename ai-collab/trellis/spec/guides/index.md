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
| [Windows 中文路径下的 Git 脚本规范](./git-scripting-on-windows.md) | 路径转义、守卫脚本自证、内容边界校验 | **在含中文路径的仓库中编写 git 自动化或提交前守卫脚本时** |

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

## When to Think About Git Scripting（Git 脚本与提交守卫）

- [ ] 正在编写解析 git 输出的脚本，而仓库中存在中文文件名或目录名
- [ ] 正在编写提交前的内容边界守卫，例如同一仓库向多个远端推送不同内容
- [ ] 守卫脚本报警但人工核对后发现内容其实正确

→ 阅读并遵循 [Windows 中文路径下的 Git 脚本规范](./git-scripting-on-windows.md)。
核心：解析路径一律加 `-c core.quotepath=false`，守卫先打印实际清单再判断，
需入库的目录不以 `.` 开头。

## When to Think About Vision MCP（图片输入处理）

- [ ] 用户输入中包含图片（截图、照片、设计稿等）
- [ ] 需要理解界面内容或提取文字

→ 阅读并遵循 [Vision MCP 调用规范](./vision-mcp-usage.md)。核心：自动调用 vision MCP，
按用户意图决定是否在 prompt 中注入工程上下文。
