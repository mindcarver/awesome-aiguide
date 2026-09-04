---
illustration_id: 20-05
type: flowchart
style: notion
generator: gpt-image-2
---

Notion 风格流程图，标题 `write 与 edit：临界区里的护栏`。左路 `writeText` 经过菱形 `createIfAbsent?`，右路 `editText` 经过 `replaceIfVersion?` 再到 `literal match?`。成功为浅蓝勾，拒绝为浅粉叉。精确标签：`createIfAbsent`、`replaceIfVersion`、`FS_STALE_VERSION`、`FS_EDIT_NOT_FOUND`。白底，清晰中文。
