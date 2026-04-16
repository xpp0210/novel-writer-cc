---
name: novel-writer
description: |
  AI小说写作引擎（子代理架构版）。当用户提到写小说、创作小说、生成网文、写故事、小说创作、网文写作、AI写作小说、
  续写章节、小说大纲、世界观设定、角色设定、小说章节生成、长篇小说创作、网络小说、玄幻/都市/科幻小说创作时触发。
  也当用户说"帮我写一本"、"继续写第X章"、"审查章节"、"生成大纲"、"创建世界观"时触发。
  覆盖从灵感到成书的完整Pipeline，支持中文长篇网文。
---

# Novel Writer v5 — AI小说写作引擎（子代理架构版）

## TLDR

1. Read `session-handoff.md` 获取下一章编号 N
2. 检查断点：已有中间文件则询问用户续传还是重来
3. 按依赖图调度子代理执行 Pipeline（可并行步骤同时发起）
4. 审计结果必须等用户确认后继续

**核心原则：主 session 只做编排，不创作不读正文。每个步骤由独立子代理执行，通过文件通信。**

**归档隔离规则**：`chapters/` 下以 `-old` 结尾的子目录（如 `v1-old/`、`v2-old/`）为归档目录，所有子代理在执行 Glob、Grep、Read 操作时必须排除所有匹配 `*-old/` 的目录。归档内容仅供人工查阅，不参与任何自动化流程。

## 架构

```
主 Session（编排器）
  │
  ├─ Step 1: Agent(constraint_extractor) → chapters/chapter-{N}-constraints.md
  │
  ├─ Step 2a+2b+2c: Agent(设定读取) + Agent(台账摘要) + Agent(近文读取)  ← 并行
  │     ├─ 2a 设定读取 → chapters/chapter-{N}-settings.json
  │     ├─ 2b 台账摘要 → chapters/chapter-{N}-ledgers.json
  │     └─ 2c 近文读取 → chapters/chapter-{N}-recent.json
  │
  ├─ Step 2d: Agent(context组装) → chapters/chapter-{N}-context.json
  │
  ├─ Step 3: Agent(writer) → chapters/chapter-{N}.md
  │
  ├─ Step 4: Agent(extractor) → chapters/chapter-{N}-extract.json
  │
  ├─ Step 5 + 6a + 6b + 6c: Agent(state_tracker) + Agent(索引更新) + Agent(伏笔更新) + Agent(冲突更新)  ← 并行
  │     ├─ state_tracker → character-state.json
  │     ├─ 6a 索引更新 → chapter-index.json
  │     ├─ 6b 伏笔更新 → 伏笔台账.json
  │     └─ 6c 冲突更新 → 冲突台账.json
  │
  ├─ Step 6d: Agent(handoff重建) → session-handoff.md
  │
  └─ Step 7+8: Agent(auditor) + Agent(logic_reviewer)  ← 并行
        ├─ auditor → chapters/chapter-{N}-audit.md
        └─ logic_reviewer → 追加到 audit.md（可选）
```

**并行规则**：Step 2a/2b/2c 无依赖可并行；Step 5/6a/6b/6c 无依赖可并行；Step 7/8 无依赖可并行。其余步骤严格串行。

## 书籍目录结构

```
~/Obsidian-Novel/<书名>/
├── 世界观.md / 角色.md / 大纲.md / 写法规则.md
├── 伏笔台账.json / 冲突台账.json / chapter-index.json / character-state.json
├── session-handoff.md
└── chapters/
    ├── chapter-NNNN.md              # 正文
    ├── chapter-NNNN-constraints.md  # 硬约束（Step 1 输出）
    ├── chapter-NNNN-settings.json  # 设定文件（Step 2a 输出）
    ├── chapter-NNNN-ledgers.json   # 台账摘要（Step 2b 输出）
    ├── chapter-NNNN-recent.json    # 近文参考（Step 2c 输出）
    ├── chapter-NNNN-context.json   # 最终上下文（Step 2d 输出）
    ├── chapter-NNNN-extract.json    # 提取JSON（Step 4 输出）
    ├── chapter-NNNN-audit.md        # 审计报告（Step 7 输出）
    └── archive-summary.md           # 归档摘要
```

数据 Schema 定义见 `references/data-schemas.md`，上下文格式见 `references/context-budget.md`。

## 执行入口判断

| 用户指令 | 执行内容 |
|---------|---------|
| "继续写" / "写下一章" / "写第N章" | 完整 Pipeline Step 1-8 |
| "创建新小说" | 初始化流程 |
| "审计第N章" | 只执行 Step 7（+8 可选） |
| "重新结算第N章" | 执行 Step 4+5+6 |
| "检查逻辑" | 触发 `/novel-logic-review` 子 skill |
| "检查设定" | 触发 `/novel-setting-audit` 子 skill |
| "润色第N章" | 触发 `/novel-chapter-polish` 子 skill |
| "检查一致性" | 触发 `/novel-setting-consistency-check` 子 skill |
| "修改第N章" | 修改 + 自动 resync（Step 4+5+6） |
| "全文重写" | 归档 + 重置 + 重新初始化 |
| "更新handoff" | 读台账 → 重建 handoff |

## 创建新小说

1. 询问用户书名、类型、核心设定概要
2. 在 `~/Obsidian-Novel/<书名>/` 创建目录和 `chapters/` 子目录
3. 初始化台账：`伏笔台账.json`、`冲突台账.json`、`chapter-index.json` 写入 `[]`
4. 初始化 `character-state.json` 写入 `{}`
5. 创建模板化的设定文件（世界观.md、角色.md、大纲.md、写法规则.md）
6. 创建 `session-handoff.md`，内容：`下一章：1`
7. 告知用户："请填写设定文件，确认后告诉我开始写作"

## 全文重写流程

1. 归档旧章：`chapters/chapter-*.md` 移到 `chapters/v{n}-old/`（归档目录匹配 `*-old/`，所有 Glob/Grep/Read 操作必须排除）
2. 重置台账：`伏笔台账.json`、`冲突台账.json`、`chapter-index.json` 写入 `[]`
3. 重置 `character-state.json` 为 `{}`
4. 重置 `session-handoff.md` 为初始状态（下一章：1）
5. 删除 `chapters/` 下所有中间文件（`-audit.md`、`-constraints.md`、`-settings.json`、`-ledgers.json`、`-recent.json`、`-context.json`、`-extract.json`）
6. 等用户确认设定文件后重新开始写作

## 断点续传

每次启动 Pipeline 前，检查中间文件判断进度：

| 检查条件 | 判断 | 操作 |
|---------|------|------|
| `chapter-{N}-constraints.md` 存在 | Step 1 已完成 | 跳过 |
| `chapter-{N}-settings.json` + `ledgers.json` + `recent.json` 都存在 | Step 2a/2b/2c 已完成 | 跳过 |
| `chapter-{N}-context.json` 存在 | Step 2d 已完成 | 跳过 |
| `chapter-{N}.md` 存在 | Step 3 已完成 | 跳过 |
| `chapter-{N}-extract.json` 存在 | Step 4 已完成 | 跳过 |
| chapter-index.json 最新条目 number=N + 台账已更新 | Step 5+6a+6b+6c 已完成 | 跳过 |
| session-handoff.md 已更新且包含第N章 | Step 6d 已完成 | 跳过 |
| `chapter-{N}-audit.md` 存在 | Step 7 已完成 | 跳过 |

展示断点状态给用户，确认后从断点继续。

## 写作 Pipeline

以下用 `{book_dir}` 表示书籍目录，`{N}` 表示章节号，`{skill_dir}` 表示本 skill 目录（`~/.claude/skills/novel-writer`）。

**子代理调度方式**：对每个步骤，Read `agents/{step}.md` 获取完整指令，将 `{book_dir}` 替换为实际书籍路径、`{N}` 替换为实际章节号、`{skill_dir}` 替换为 `~/.claude/skills/novel-writer`，然后作为 Agent 工具的 prompt 传入（subagent_type="general-purpose"）。

### Step 1: constraint_extractor

Agent 文件：`agents/01-constraint-extractor.md`
输入：世界观.md、大纲.md、角色.md、写法规则.md
输出：`chapters/chapter-{N:04d}-constraints.md`

### Step 2: 上下文组装（4个子代理，2a/2b/2c 并行 → 2d 串行）

将原来单个 composer 拆分为 3 个并行读取子代理 + 1 个最终组装子代理，避免单次读取过多文件。

#### Step 2a: 设定读取（并行）

Agent 文件：`agents/02a-settings-reader.md`
输出：`chapters/chapter-{N:04d}-settings.json`

#### Step 2b: 台账摘要（并行）

Agent 文件：`agents/02b-ledgers-reader.md`
输出：`chapters/chapter-{N:04d}-ledgers.json`

#### Step 2c: 近文读取（并行）

Agent 文件：`agents/02c-recent-reader.md`
输出：`chapters/chapter-{N:04d}-recent.json`

#### Step 2d: 最终组装（依赖 2a+2b+2c 全部完成）

Agent 文件：`agents/02d-context-assembler.md`
输出：`chapters/chapter-{N:04d}-context.json`

### Step 3: writer

Agent 文件：`agents/03-writer.md`
输入：Step 2d 输出的 context.json
输出：`chapters/chapter-{N:04d}.md`

### Step 4: extractor

Agent 文件：`agents/04-extractor.md`
输入：Step 3 输出的 chapter.md
输出：`chapters/chapter-{N:04d}-extract.json`

### Step 5 + 6a + 6b + 6c: 并行执行

Step 4 完成后，同时发起四个 Agent 调用。

#### Step 5: state_tracker

Agent 文件：`agents/05-state-tracker.md`
输出：`character-state.json`（覆盖写入）

#### Step 6a: 索引更新

Agent 文件：`agents/06a-index-updater.md`
输出：`chapter-index.json`（覆盖写入）

#### Step 6b: 伏笔台账更新

Agent 文件：`agents/06b-foreshadow-updater.md`
输出：`伏笔台账.json`（覆盖写入）

#### Step 6c: 冲突台账更新

Agent 文件：`agents/06c-conflict-updater.md`
输出：`冲突台账.json`（覆盖写入）

### Step 6d: handoff 重建（依赖 Step 5 + 6a + 6b + 6c 全部完成）

Agent 文件：`agents/06d-handoff-builder.md`
输出：`session-handoff.md`

### Step 7 + Step 8: 并行执行

Step 6d 完成后，同时发起两个 Agent 调用。

#### Step 7: auditor

Agent 文件：`agents/07-auditor.md`
输出：`chapters/chapter-{N:04d}-audit.md`

#### Step 8: logic_reviewer（可选，章节>3时执行）

Agent 文件：`agents/08-logic-reviewer.md`
输出：追加到 `chapters/chapter-{N:04d}-audit.md` 末尾

## 用户确认点（不可跳过）

所有子代理完成后，主 session 展示审计结果：

| 场景 | 操作 |
|------|------|
| 审计通过 | 展示结论+字数+摘要，问"确认发布？" |
| 需小修 | 展示问题，问"自动修复还是手动改？" |
| 需重写 | 展示原因，问"重写还是调大纲？" |

## 章节修改与同步

任何导致正文内容变化的操作，完成后必须执行 resync（详见 `references/sync-protocol.md`）：

1. 发起 Step 4(extractor) 子代理
2. Step 4 完成后，并行发起 Step 5(state_tracker) + Step 6a(索引) + Step 6b(伏笔) + Step 6c(冲突) 子代理
3. 上述全部完成后，发起 Step 6d(handoff重建) 子代理
4. 删除旧审计报告
5. 大段重写（>30% 内容变动）→ 重新发起 Step 7+8；小幅修补 → 跳过

## session-handoff.md 规则

必须列出**全部**活跃伏笔和冲突（status≠已兑现/已关闭），不做选择性删减。

**同步检查**（每次写新章前都要验证）：
1. 伏笔条数 = `伏笔台账.json` 中 status=活跃 的条数
2. 冲突条数 = `冲突台账.json` 中 status=活跃 的条数
3. 概要无重复前缀
4. 概要从 chapter-index.json 的 key_events 同步
5. 冲突条目如有 expected_resolution，在 handoff 中标注

如果条目数不一致，从台账重建 handoff。

## 错误处理

| 场景 | 处理 |
|------|------|
| 子代理输出文件不存在 | 检查该步骤是否成功执行，重新发起该子代理 |
| 子代理输出格式错误 | 重新发起该子代理，在 prompt 中强调输出格式要求 |
| 台账 JSON 损坏 | 重新发起 extractor + settler |
| 台账去重遗漏 | settler 完成后用 Grep 检查重复条目 |
| 世界观过大淹没关键规则 | constraint_extractor 应在顶部单独标注关键规则 |

## 参考文件索引

| 文件 | 用途 |
|------|------|
| `agents/` | 各步骤子代理指令（14个独立文件） |
| `references/data-schemas.md` | 所有 JSON 数据结构定义（含中间文件） |
| `references/context-budget.md` | 上下文裁剪策略 + context.json 格式 |
| `references/audit-checklist.md` | 12 项审计 + 4 类逻辑审查完整清单 |
| `references/sync-protocol.md` | resync 协议 + 台账健康检查 |
| `shared/writing-rules.md` | 默认写法规则 |
| `shared/banned-words.md` | 禁用词列表 |
