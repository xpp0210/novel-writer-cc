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
  ├─ Step 5+6: Agent(state_tracker) + Agent(settler)  ← 并行
  │     ├─ state_tracker → character-state.json
  │     └─ settler → chapter-index.json + 台账 + session-handoff.md
  │
  └─ Step 7+8: Agent(auditor) + Agent(logic_reviewer)  ← 并行
        ├─ auditor → chapters/chapter-{N}-audit.md
        └─ logic_reviewer → 追加到 audit.md（可选）
```

**并行规则**：Step 2a/2b/2c 无依赖可并行；Step 5/6 无依赖可并行；Step 7/8 无依赖可并行。其余步骤严格串行。

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

1. 归档旧章：`chapters/chapter-*.md` 移到 `chapters/v1-old/`
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
| chapter-index.json 最新条目 number=N | Step 5+6 已完成 | 跳过 |
| `chapter-{N}-audit.md` 存在 | Step 7 已完成 | 跳过 |

展示断点状态给用户，确认后从断点继续。

## 写作 Pipeline

以下用 `{book_dir}` 表示书籍目录，`{N}` 表示章节号，`{skill_dir}` 表示 `~/.claude/skills/novel-writer`。

### Step 1: constraint_extractor（子代理）

**使用 Agent 工具**，subagent_type="general-purpose"，prompt 如下：

```
你是约束提取专家。任务：从世界观中提取第{N}章相关的硬约束。

操作步骤：
1. Read {book_dir}/世界观.md
2. Read {book_dir}/大纲.md — 找到包含 | {N} | 的行，了解本章核心事件
3. Read {book_dir}/角色.md
4. Read {book_dir}/写法规则.md（如不存在则 Read {skill_dir}/shared/writing-rules.md）
5. 判断本章涉及哪些规则（跨阶战斗？面板能力分层？系别相性？特定角色设定？）
6. 从世界观中提取相关段落，压缩为≤300字的"本章硬约束"

提取原则：
- 只提取本章事件直接涉及的规则
- 优先提取"容易违反"的规则（跨阶限制、系别相性）
- 矛盾双方同时提取（大纲vs世界观）
- 面板能力分层展示规则

7. Write {book_dir}/chapters/chapter-{N:04d}-constraints.md，写入提取的硬约束文本
```

### Step 2: 上下文组装（4个子代理，2a/2b/2c 并行 → 2d 串行）

将原来单个 composer 拆分为 3 个并行读取子代理 + 1 个最终组装子代理，避免单次读取过多文件。

#### Step 2a: 设定读取（子代理）

**使用 Agent 工具**，subagent_type="general-purpose"，prompt 如下：

```
你是设定文件读取专家。任务：读取世界观、角色、大纲，提取第{N}章相关内容。

操作步骤：
1. Read {book_dir}/世界观.md → 全文保留。如果超过50K字符，按段落边界裁剪，保留标题行和表格行优先
2. Read {book_dir}/角色.md → 全文保留
3. Read {book_dir}/大纲.md → 用 Grep 提取包含 | {N} | 的行及前后3行，≤800字
4. Read {book_dir}/写法规则.md（如不存在则 Read {skill_dir}/shared/writing-rules.md）→ 只保留以#、-、*、❌、✅开头的行，≤400字

Write {book_dir}/chapters/chapter-{N:04d}-settings.json：
{
  "world": "世界观全文（或裁剪后）",
  "characters": "角色全文",
  "outline": "本章大纲段落（≤800字）",
  "writing_rules": "写法规则（≤400字）"
}
```

#### Step 2b: 台账摘要（子代理）

**使用 Agent 工具**，subagent_type="general-purpose"，prompt 如下：

```
你是台账数据摘要专家。任务：读取所有台账和索引文件，提取第{N}章需要的摘要信息。

操作步骤：
1. Read {book_dir}/chapter-index.json → 提取最近15章摘要（距当前章>5只保留number+title；3-5保留summary；≤2保留完整）
2. Read {book_dir}/伏笔台账.json → 筛选status≠已兑现/已关闭的条目，取前10条，每条content截断到50字
3. Read {book_dir}/冲突台账.json → 筛选status≠已解决的条目，取前6条，每条content截断到50字
4. Read {book_dir}/character-state.json → 每人保留最近2条changes

Write {book_dir}/chapters/chapter-{N:04d}-ledgers.json：
{
  "chapter_summaries": [{"num": 1, "title": "标题", "summary": "摘要"}],
  "active_foreshadow": [{"id": "f001", "content": "伏笔内容", "planted_chapter": 3}],
  "active_conflicts": [{"id": "c001", "content": "冲突内容", "introduced_chapter": 2}],
  "character_state": {"角色名": {"last_seen": 4, "changes": ["..."]}}
}
```

#### Step 2c: 近文读取（子代理）

**使用 Agent 工具**，subagent_type="general-purpose"，prompt 如下：

```
你是近文参考读取专家。任务：读取前几章正文，提取衔接所需的文本片段。

操作步骤：
1. Read {book_dir}/chapters/chapter-{N-1:04d}.md → 如果文件存在，取后半部分2000字+末尾500字。如果文件不存在（N=1），recent_text 留空
2. Read {book_dir}/chapters/chapter-{N-2:04d}.md → 如果文件存在，取末尾500字。不存在则跳过
3. Read {book_dir}/chapters/chapter-{N-3:04d}.md → 如果文件存在，取末尾500字。不存在则跳过

Write {book_dir}/chapters/chapter-{N:04d}-recent.json：
{
  "recent_text": "前一章后半2000字+末尾500字（N=1时为空字符串）",
  "prev_chapters": [
    {"num": {N-2}, "tail": "末尾500字"},
    {"num": {N-3}, "tail": "末尾500字"}
  ]
}
prev_chapters 数组只包含实际存在且成功读取的章节，不要放入空内容。
```

#### Step 2d: 最终组装（子代理，依赖 2a+2b+2c 全部完成）

**使用 Agent 工具**，subagent_type="general-purpose"，prompt 如下：

```
你是上下文组装专家。任务：读取 Step 2a/2b/2c 的输出 + 共享资源，组装为 writer 可用的最终上下文。

操作步骤：
1. Read {book_dir}/chapters/chapter-{N:04d}-settings.json（Step 2a 输出）
2. Read {book_dir}/chapters/chapter-{N:04d}-ledgers.json（Step 2b 输出）
3. Read {book_dir}/chapters/chapter-{N:04d}-recent.json（Step 2c 输出）
4. Read {skill_dir}/shared/banned-words.md → 全文保留
5. Read {book_dir}/chapters/chapter-{N:04d}-constraints.md（Step 1 输出）

合并为最终上下文 JSON，Write 到 {book_dir}/chapters/chapter-{N:04d}-context.json：
{
  "chapter": {N},
  "output_path": "{book_dir}/chapters/chapter-{N:04d}.md",
  "constraints": "本章硬约束文本",
  "writing_rules": "写法规则文本",
  "banned_words": "禁用词列表全文",
  "world": "世界观全文",
  "characters": "角色全文",
  "character_state": {...从ledgers.json取...},
  "outline": "本章大纲段落",
  "recent_text": "前一章后半+末尾",
  "prev_chapters": [...从recent.json取...],
  "chapter_summaries": [...从ledgers.json取...],
  "active_foreshadow": [...从ledgers.json取...],
  "active_conflicts": [...从ledgers.json取...]
}

直接合并，不再做额外裁剪（裁剪已在 2a/2b/2c 中完成）。
```

### Step 3: writer（子代理）

**使用 Agent 工具**，subagent_type="general-purpose"，prompt 如下：

```
你是网文写手。任务：基于组装好的上下文创作第{N}章正文。

操作步骤：
1. Read {book_dir}/chapters/chapter-{N:04d}-context.json
2. 基于上下文创作3000-5000字正文
3. Write {book_dir}/chapters/chapter-{N:04d}.md

严格约束：
- 不读任何其他文件，只基于 context.json 中的内容
- 不在正文中混入英文单词
- 禁用词零触发（参考 context 中的 banned_words 字段）
- 面板数据用中文格式（键：值，| 分隔）
- 章末必须留钩子（悬念/转折/伏笔暗示/新信息）
- 前100字必须进入场景，不要用大段背景交代开场
- 对话要有口语感，标签多样
- 描写要具体，避免空泛形容词
- 情感通过动作/环境间接表达
```

### Step 4: extractor（子代理）

**使用 Agent 工具**，subagent_type="general-purpose"，prompt 如下：

```
你是信息提取专家。任务：从第{N}章正文中提取结构化信息。

操作步骤：
1. Read {book_dir}/chapters/chapter-{N:04d}.md
2. 提取以下结构化信息
3. Write {book_dir}/chapters/chapter-{N:04d}-extract.json

输出格式（严格 JSON，不要包含其他内容）：
{
  "title": "章节标题",
  "word_count": 正文字数,
  "summary": "50-100字章节摘要",
  "key_events": ["事件1", "事件2", ...],
  "character_changes": ["变化1", "变化2", ...],
  "foreshadowing_activity": ["伏笔活动1", ...],
  "conflicts": ["冲突1", ...]
}

字段约束：key_events≤5，character_changes≤4，foreshadowing_activity≤4，conflicts≤3。
title 从正文第一个标题或第一段推断。
word_count 统计正文中文字符数（不含标点和空格）。
```

### Step 5 + Step 6: 并行执行

**同时发起两个 Agent 调用**：

#### Step 5: state_tracker（子代理）

```
你是角色状态追踪专家。任务：识别第{N}章中角色的实质性状态变化。

操作步骤：
1. Read {book_dir}/chapters/chapter-{N:04d}.md（正文）
2. Read {book_dir}/character-state.json（当前状态）
3. Read {book_dir}/角色.md（角色设定）

判断哪些角色有实质性状态变化：
- 只记录关系/实力/知晓秘密/立场等变化
- 被顺带提到不算（如"张三也在场"不记录）
- 实力变化要具体（如"F级→E级"而非"变强了"）

4. Write {book_dir}/character-state.json（覆盖写入，格式如下）：
{
  "角色名": {
    "last_seen": {N},
    "changes": [
      "第X章：之前的变化",
      "第{N}章：本次变化"
    ]
  }
}
注意：保留该角色之前的 changes 记录，只追加新变化。未出场的角色保持原样。
```

#### Step 6: settler（子代理）

```
你是章节结算专家。任务：基于 extractor 的输出更新所有台账文件。

操作步骤：
1. Read {book_dir}/chapters/chapter-{N:04d}-extract.json（extractor 输出的结构化 JSON）

#### 6.1 更新 chapter-index.json
Read {book_dir}/chapter-index.json
- 如果已存在 number={N} 的条目，删除旧条目
- 追加新条目：{number: N, title, status: "已通过", word_count, summary, key_events(≤5), character_state_changes(≤4), foreshadowing_activity(≤4)}
- 摘要瘦身：距当前章>5只保留number+title（详情追加到chapters/archive-summary.md）；3-5只保留summary；≤2保留完整；总大小≤5000字节
Write {book_dir}/chapter-index.json

#### 6.2 更新伏笔台账.json
Read {book_dir}/伏笔台账.json
- 获取已有最大ID：扫描所有 f\d+ 格式的ID
- 遍历 extract.json 的 foreshadowing_activity：
  - 语义去重：判断是否与已有条目语义重复
  - 不重复则分配新ID f{max+1:03d}
  - 追加：{"id": "fXXX", "content": "内容", "planted_chapter": N, "status": "活跃"}
Write {book_dir}/伏笔台账.json

#### 6.3 更新冲突台账.json
Read {book_dir}/冲突台账.json
- 同上逻辑，ID前缀为 c
Write {book_dir}/冲突台账.json

#### 6.4 更新 session-handoff.md
Read 所有台账获取活跃条目
Read chapter-index.json 获取最近3章概要
Read character-state.json 获取角色状态
按以下模板 Write {book_dir}/session-handoff.md：
# Session Handoff

## 当前进度
第{N}章已完成，下一章编号：{N+1}
标题：{title}

## 最近3章概要
- 第X章「标题」摘要

## 角色当前状态
- 角色名(第X章): 变化1; 变化2

## 关键待处理伏笔
- f001(伏笔内容)

## 关键待处理冲突
- c001(冲突内容)

## 台账状态
伏笔X条活跃 / 冲突Y条活跃

#### 6.5 台账瘦身（每5章执行一次）
如果 {N} 是5的倍数：
- 遍历伏笔台账：status=已兑现→删除note；status=已推进且距planted_chapter>10→删除note；其他活跃→note截断到80字
- 对冲突台账执行相同操作
- 重新 Write 两个台账文件
```

### Step 7 + Step 8: 并行执行

**同时发起两个 Agent 调用**：

#### Step 7: auditor（子代理）

```
你是章节审计专家。任务：对第{N}章执行12项质量检查。

操作步骤：
1. Read {book_dir}/chapters/chapter-{N:04d}.md（本章正文）
2. Read {book_dir}/chapters/chapter-{N-1:04d}.md（前一章，如存在）
3. Read {book_dir}/世界观.md
4. Read {book_dir}/角色.md
5. Read {book_dir}/伏笔台账.json
6. Read {book_dir}/冲突台账.json
7. Read {skill_dir}/shared/banned-words.md

12项必检：
1. 第四面墙：正文不得出现章节数、角色不知道的叙事信息
2. 跨章一致性：关键设定与前章描写是否矛盾（用Grep搜索验证）
3. OOC：角色言行是否符合人设
4. 时间线：与前后章时间是否矛盾
5. 设定一致性：是否违反世界观硬约束（重点：异能视觉效果）
6. 大纲覆盖度：是否覆盖大纲中该章所有核心事件
7. 伏笔处理：应兑现的是否兑现，新伏笔是否合理
8. 文风：禁用词零触发、空泛形容词零触发、AI痕迹零触发（Grep扫描）
9. 重复：与前后章是否有重复描写
10. 中英混用：正文是否有不当英文
11. 台账准确性：台账描述与正文是否一致，有无遗漏
12. 叙事节奏：前100字进入场景、冲突位置合理、无超300字低密度段落、章末有钩子

输出格式：
## 审计结果：✅通过 / ⚠️需小修 / ❌需重写

### 📊 基础数据
- 标题：...
- 字数：...
- 摘要：...

### 🔴 必须修
| # | 类型 | 问题 | 建议 |

### 🟡 建议修
| # | 类型 | 问题 | 建议 |

### ⚠️ 需确认
| # | 类型 | 问题 | 建议 |

Write {book_dir}/chapters/chapter-{N:04d}-audit.md
```

#### Step 8: logic_reviewer（子代理，章节>3时执行）

```
你是跨章逻辑审查专家。任务：检查第{N}章与前文的逻辑一致性。

操作步骤：
1. Read {book_dir}/世界观.md
2. Read {book_dir}/角色.md
3. Read {book_dir}/大纲.md
4. Read {book_dir}/chapter-index.json
5. Read {book_dir}/chapters/chapter-{N:04d}.md
6. Read {book_dir}/chapters/chapter-{N-1:04d}.md（前一章）
7. Read {book_dir}/chapters/chapter-{N-2:04d}.md（前两章，如存在）

4类检查：
1. 悬空描述：引用了未在上下文中出现过的信息
2. 时间线矛盾：跨章事件时间冲突、距离/时间估算不合理
3. 设定矛盾：违反世界观硬规则、角色能力/状态前后矛盾
4. 信息前后不一致：同一事件不同描述、角色名/称呼不一致、数字不一致

跨章验证（必须执行）：
用 Grep 工具搜索 chapters/ 目录，验证以下关键词在所有章节中的一致性：
- 本章出现的角色名（搜等级/称呼/状态）
- 本章涉及的异能名称（搜能力描写）
- 本章提到的数字/编号
- 本章涉及的时间表述

发现匹配后读取相关章节片段交叉对比。

输出格式（追加到审计报告）：
## 逻辑审查结果

### 🔴 必须修的硬伤
| # | 章节 | 类型 | 引用原文 | 原因 |

### 🟡 建议修的
| # | 章节 | 类型 | 引用原文 | 原因 |

### ⚠️ 需确认
| # | 章节 | 类型 | 引用原文 | 原因 |

将结果追加到 {book_dir}/chapters/chapter-{N:04d}-audit.md 末尾（Read已有内容再追加写入）。
```

## 用户确认点（不可跳过）

所有子代理完成后，主 session 展示审计结果：

| 场景 | 操作 |
|------|------|
| 审计通过 | 展示结论+字数+摘要，问"确认发布？" |
| 需小修 | 展示问题，问"自动修复还是手动改？" |
| 需重写 | 展示原因，问"重写还是调大纲？" |

## 章节修改与同步

任何导致正文内容变化的操作，完成后必须执行 resync（详见 `references/sync-protocol.md`）：

1. 发起 Step 4(extractor) + Step 5(state_tracker) + Step 6(settler) 子代理
2. 删除旧审计报告
3. 大段重写（>30% 内容变动）→ 重新发起 Step 7+8；小幅修补 → 跳过

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
| `references/data-schemas.md` | 所有 JSON 数据结构定义（含中间文件） |
| `references/context-budget.md` | 上下文裁剪策略 + context.json 格式 |
| `references/audit-checklist.md` | 12 项审计 + 4 类逻辑审查完整清单 |
| `references/sync-protocol.md` | resync 协议 + 台账健康检查 |
| `shared/writing-rules.md` | 默认写法规则 |
| `shared/banned-words.md` | 禁用词列表 |
