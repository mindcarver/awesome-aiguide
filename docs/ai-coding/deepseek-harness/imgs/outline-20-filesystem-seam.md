---
article: 20-filesystem-seam.md
style: notion
image_count: 16
generator: gpt-image-2
---

# Filesystem 接缝插图大纲

1. `ctx.fs` 将 provider、策略和消费者解耦。
2. 不透明目标身份与原子文件操作契约。
3. 版本令牌把“看见”变成可验证的新鲜度。
4. 完整读取、窗口读取与不存在三种观察结果。
5. `writeText` 与 `editText` 的临界区护栏。
6. 缺席也是记录在案的观察状态。
7. 从 v7 到删除再创建的典型竞态。
8. read-before-edit 作为事件策略层，而非 provider 内置逻辑。
9. 共享词汇表：发射方与监听方独立演进。
10. 局部读取携带版本令牌即可授权编辑。
11. 文件系统与 bash 共享沙箱和审批边界。
12. `str_replace_editor` 是同一策略下的另一位消费者。
13. 文件 IO 的取消点在发布前，而不是任意超时。
14. 稳定错误码把失败转成可处理语义。
15. 原子发布避免 TOCTOU 的“不覆盖”承诺。
16. 能力、策略、观察和执行共同组成文件系统接缝。
