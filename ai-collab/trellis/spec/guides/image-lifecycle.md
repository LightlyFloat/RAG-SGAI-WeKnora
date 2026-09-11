# 图片生命周期管理规范

> **适用范围**：使用 Claude Code 进行开发时，会话内存在历史图片（截图、设计稿、照片等）的情况。

---

## 核心规则

会话结束时，`SessionEnd` hook 自动将历史轮次的图片降级为文本占位符，当前轮图片保持完整。所有被降级的图片均按内容哈希归档至本地存储，可随时还原，无数据损失。

---

## 为什么需要管理图片生命周期

Claude Code 将会话中每张图片以 base64 格式存储在 transcript 文件中，且每轮请求均会将所有历史图片完整重发给模型。随着会话延长：

- 上下文窗口快速膨胀，压缩机制（`/compact`）难以跟上
- Prompt cache 前缀不稳定，命中率下降
- 请求体过大时触发 413 错误，导致会话终止

---

## 自动处理机制

本项目已在 `.claude/settings.json` 中配置 `SessionEnd` hook：

```json
{
  "hooks": {
    "SessionEnd": [
      {
        "matcher": "",
        "hooks": [
          { "type": "command", "command": "image-cascade hook claude-code --store" }
        ]
      }
    ]
  }
}
```

**行为说明：**

| 时机 | 行为 |
|------|------|
| 会话结束时 | 自动扫描 transcript，将历史图片降级为稳定占位符 |
| 当前轮图片 | 保持完整，不受影响 |
| 历史图片 | 降级为文本占位符，原始数据归档至 `~/.image-cascade/store` |

hook 执行耗时为毫秒级，不影响会话响应速度。无常驻进程，不占用系统资源。

---

## 手动操作

### 对历史会话一次性压缩

```bash
npx @image-cascade/cli rescue <session.jsonl> --yes --store
```

不加 `--yes` 时为 dry-run 模式，仅输出统计信息，不修改文件。

### 还原被降级的图片

```bash
image-cascade restore <hash>
```

`hash` 值来自占位符文本或 `--json` 输出中的 `hashes` 字段。

---

## 前置依赖

每台开发机器需全局安装一次：

```bash
npm install -g @image-cascade/cli
```

`.claude/settings.json`（含 hook 配置）已提交至 git，`git pull` 后各机器自动获得配置，无需重复设置。

---

## 使用检查清单

- [ ] 本机已执行 `npm install -g @image-cascade/cli`
- [ ] `.claude/settings.json` 中 `SessionEnd` hook 已配置
- [ ] 存量历史会话已通过 `rescue` 命令处理

---

## 工具参考

- 项目地址：`https://github.com/dlgod7/image-context-cascade`
- CLI 包：`@image-cascade/cli`
- 核心包：`image-context-cascade`
