---
type: mixed
density: rich
style: notion
palette: default
image_count: 10
article: 12-capability-seams-swap-provider-swap-product.md
---

# 能力接缝：换 provider 等于换产品 — 配图大纲

## Illustration 1
**Position**: 接缝三角色与 seam/core/bundle 分类之后
**Purpose**: 建立能力接缝必须由定义、提供者、消费者共同组成的心智模型。
**Visual Content**: 三角色流向同一可替换能力，旁列三种服务角色。
**Text Plan**: 标题「能力接缝的三个角色」；标签「Service Definition」「Service Provider」「Consumer」「seam / core / bundle」；提示「三件齐备才是接缝」。
**Filename**: 12-01-framework-seam-roles.png

## Illustration 2
**Position**: 共享执行世界推演之后
**Purpose**: 显示 fs 与 subprocess 一同切换到远程 E2B 后带走的消费者。
**Visual Content**: fs-e2b 与 subprocess-e2b 汇入同一远程 Linux 运行时，连接 Bash、PTY、LSP、子 agent。
**Text Plan**: 标题「共享执行世界」；标签「fs-e2b」「subprocess-e2b」「远程 Linux」「Bash / PTY / LSP」；提示「换两个 provider，搬走整个执行世界」。
**Filename**: 12-02-framework-shared-execution-world.png

## Illustration 3
**Position**: ctx.llm 小节之后
**Purpose**: 展现模型适配器如何在稳定 stream 接口后替换 provider。
**Visual Content**: 多个 llm provider 连入 provider-neutral stream，再连向 agent-loop 和 compaction。
**Text Plan**: 标题「ctx.llm：稳定接口，替换模型」；标签「llm-deepseek」「llm-pi-ai」「llm-replay」「agent-loop」；提示「消费者不 import 具体实现」。
**Filename**: 12-03-framework-llm-adapters.png

## Illustration 4
**Position**: ctx.fs 小节之后
**Purpose**: 对比本地、围栏和远程文件系统 provider 的相同工具消费者。
**Visual Content**: fs-local、fs-sandbox、fs-e2b 三选一连接 tool-fs，围栏强调共享 sandbox policy。
**Text Plan**: 标题「ctx.fs：文件系统可替换」；标签「fs-local」「fs-sandbox」「fs-e2b」「tool-fs」；提示「工具接口不变，执行位置可换」。
**Filename**: 12-04-comparison-fs-providers.png

## Illustration 5
**Position**: ctx.subprocess 小节之后
**Purpose**: 说明 subprocess 是 Bash、终端、LSP、进程外子 agent 的共同枢纽。
**Visual Content**: subprocess 中心卡连接 bash、PTY、LSP、ACP/Codex/Claude 子 agent。
**Text Plan**: 标题「ctx.subprocess：执行世界枢纽」；标签「Bash」「PTY」「LSP」「进程外子 agent」；提示「所有 spawn 都走同一个接缝」。
**Filename**: 12-05-framework-subprocess-hub.png

## Illustration 6
**Position**: sandbox 与 sandboxPolicy 小节之后
**Purpose**: 表达 bash 和 fs 必须共享同一 workspace root 的一致性约束。
**Visual Content**: sandboxPolicy 中心 root 卡同时约束 bash sandbox 与 fs sandbox，错误分支显示不同 root 会泄漏。
**Text Plan**: 标题「同一个 root，才是同一个围栏」；标签「sandboxPolicy」「bash-sandbox」「fs-sandbox」「workspace root」；提示「不同 root 会造成沙箱泄漏」。
**Filename**: 12-06-flowchart-sandbox-root-consistency.png

## Illustration 7
**Position**: ctx.subagents 小节之后
**Purpose**: 表达一个统一接口背后可以是进程内新 agent 或其他产品的一次委派。
**Visual Content**: tool-subagent 接口分支到 in-process、ACP、Codex、Claude Code、SDK。
**Text Plan**: 标题「ctx.subagents：委派形态可换」；标签「in-process」「ACP」「Codex」「Claude Code」；提示「同一接口，背后可以是另一个产品」。
**Filename**: 12-07-framework-subagent-providers.png

## Illustration 8
**Position**: approval 与 permissionPresets 小节之后
**Purpose**: 突出回答者缺席时权限决策 fail closed，而非默认放行。
**Visual Content**: approval/request waterfall 问询，回答者存在则决定，缺席则到 unavailable 拒绝卡。
**Text Plan**: 标题「审批缺席，默认失败关闭」；标签「approval/request」「回答者」「unavailable」「fail closed」；提示「没人作答，不等于放行」。
**Filename**: 12-08-flowchart-approval-fail-closed.png

## Illustration 9
**Position**: 三个具体推演之后
**Purpose**: 汇总换 provider 如何改变执行环境、模型与协作形态。
**Visual Content**: 三组 before/after provider 交换：E2B、LLM、subagent Claude Code。
**Text Plan**: 标题「换 provider，变的是产品形态」；标签「远程执行」「换模型」「换委派形态」「消费者代码不变」；提示「替换发生在 provider」。
**Filename**: 12-09-comparison-provider-product-shifts.png

## Illustration 10
**Position**: seam/core/bundle 完整清单之后
**Purpose**: 清晰区分设计欢迎替换的 seam、稳定脊柱 core 和唯一组合点 bundle。
**Visual Content**: 三列服务卡，分别列 ctx.llm/fs、ctx.sessions/tools、ctx.agentLoop。
**Text Plan**: 标题「seam、core、bundle 的边界」；标签「可替换 seam」「核心脊柱 core」「唯一组合点 bundle」「agentLoop」；提示「能换什么，由服务角色决定」。
**Filename**: 12-10-comparison-seam-core-bundle.png
