---
name: novel-writer
description: |
  AI小说写作引擎（纯指令版）。当用户提到写小说、创作小说、生成网文、写故事、小说创作、网文写作、AI写作小说、
  续写章节、小说大纲、世界观设定、角色设定、小说章节生成、长篇小说创作、网络小说、玄幻/都市/科幻小说创作时触发。
  也当用户说"帮我写一本"、"继续写第X章"、"审查章节"、"生成大纲"、"创建世界观"时触发。
  覆盖从灵感到成书的完整Pipeline，支持中文长篇网文。
---

# Novel Writer v4 — AI小说写作引擎（纯指令版）

## TLDR

1. Read `session-handoff.md` 获取下一章编号 N
2. 串行执行 7 步 Pipeline：约束提取 → 上下文组装 → 写正文 → 摘要提取 → 角色状态追踪 → 结算 → 审计
3. 审计结果必须等用户确认后继续

**核心原则：不依赖任何 Python 脚本，所有数据操作由 Claude 原生工具（Read/Write/Grep/Glob/Bash）完成。**

## 书籍目录结构

```
~/Obsidian-Novel/<书名>/
├── 世界观.md / 角色.md / 大纲.md / 写法规则.md
├── 伏笔台账.json / 冲突台账.json / chapter-index.json / character-state.json
├── session-handoff.md
└── chapters/
    ├── chapter-NNNN.md
    ├── chapter-NNNN-audit.md
    ├── chapter-NNNN-constraints.md
    └── archive-summary.md
```

数据 Schema 定义见 `references/data-schemas.md`。

## 执行入口判断

| 用户指令 | 执行内容 |
|---------|---------|
| "继续写" / "写下一章" / "写第N章" | 完整 Pipeline Step 1-7 |
| "创建新小说" | 初始化流程 |
| "审计第N章" | 只执行 Step 7 |
| "重新结算第N章" | 执行 Step 4+5+6 |
| "检查逻辑" | 触发 `/novel-logic-review` 子 skill |
| "检查设定" | 触发 `/novel-setting-audit` 子 skill |
| "润色第N章" | 触发 `/novel-chapter-polish` 子 skill |
| "检查一致性" | 触发 `/novel-setting-consistency-check` 子 skill |
| "修改第N章" | 修改 + 自动 resync |
| "全文重写" | 归档 + 重置 + 重新初始化 |
| "更新handoff" | 读台账 → 重建 handoff |

## 创建新小说

1. 询问用户书名、类型、核心设定概要
2. 在 `~/Obsidian-Novel/<书名>/` 创建目录
3. 创建子目录 `chapters/`
4. 初始化 JSON 台账：`伏笔台账.json`、`冲突台账.json`、`chapter-index.json` 写入 `[]`
5. 初始化 `character-state.json` 写入 `{}`
6. 创建模板化的设定文件（世界观.md、角色.md、大纲.md、写法规则.md）
7. 创建 `session-handoff.md`，内容：`下一章：1`
8. 告知用户："请填写设定文件，确认后告诉我开始写作"

## 全文重写流程

1. 归档旧章：`chapters/chapter-*.md` 移到 `chapters/v1-old/`
2. 重置台账：`伏笔台账.json`、`冲突台账.json`、`chapter-index.json` 写入 `[]`
3. 重置 `character-state.json` 为 `{}`
4. 重置 `session-handoff.md` 为初始状态（下一章：1）
5. 删除 `chapters/` 下所有 `-audit.md` 和 `-constraints.md` 文件
6. 等用户确认设定文件后重新开始写作

## 写作 Pipeline

### Step 1: 约束提取

**目的**：从世界观中提取本章相关的硬约束，防止关键规则被大量细节淹没。

**操作**：
1. Read `{book_dir}/世界观.md`
2. Read `{book_dir}/大纲.md`
3. Read `{book_dir}/角色.md`
4. Read `{book_dir}/写法规则.md`（如不存在则 Read skill 目录下的 `shared/writing-rules.md`）
5. 判断本章涉及哪些规则（跨阶战斗？面板能力分层？系别相性？特定角色设定？）
6. 从世界观中提取相关段落，压缩为 ≤300 字的"本章硬约束"
7. Write `{book_dir}/chapters/chapter-{N:04d}-constraints.md`

**提取原则**：
- 只提取本章事件直接涉及的规则
- 优先提取"容易违反"的规则（跨阶限制、系别相性）
- 矛盾双方同时提取（大纲 vs 世界观）
- 面板能力分层展示规则

### Step 2: 上下文组装

**目的**：读取所有数据文件，按预算裁剪，组装为写作 context。

**操作**：用 Read 工具依次读取以下文件，按预算裁剪后组装（裁剪策略见 `references/context-budget.md`）：

1. Read `{book_dir}/世界观.md` → 全文保留（通常 <25K 字符，超 50K 按段落裁剪）
2. Read `{book_dir}/角色.md` → 全文保留
3. Read `{book_dir}/大纲.md` → 用 Grep 提取包含 `| {N} |` 的行及前后 3 行，≤800 字
4. Read `{book_dir}/写法规则.md`（如不存在则 Read `~/.claude/skills/novel-writer/shared/writing-rules.md`）→ 只保留标题行+列表行，≤400 字
5. Read `~/.claude/skills/novel-writer/shared/banned-words.md` → 全文保留
6. Read `{book_dir}/chapter-index.json` → 提取最近 15 章摘要
7. Read `{book_dir}/伏笔台账.json` → 筛选 status≠已兑现/已关闭 的条目，取前 10 条，content 截断到 50 字
8. Read `{book_dir}/冲突台账.json` → 筛选 status≠已解决 的条目，取前 6 条，content 截断到 50 字
9. Read `{book_dir}/character-state.json` → 每人保留最近 2 条 changes
10. Read `{book_dir}/chapters/chapter-{N-1:04d}.md` → 取后半部分 2000 字 + 末尾 500 字（N=1 时跳过）
11. Read `{book_dir}/chapters/chapter-{N-2:04d}.md` → 取末尾 500 字（N≤2 时跳过）
12. Read `{book_dir}/chapters/chapter-{N-3:04d}.md` → 取末尾 500 字（N≤3 时跳过）
13. Read `{book_dir}/chapters/chapter-{N:04d}-constraints.md`（Step 1 生成）

**组装顺序**（前面的内容 writer 更容易关注到）：
1. 本章硬约束
2. 写作规则
3. 禁用词
4. 世界观
5. 角色
6. 角色当前状态
7. 本章大纲
8. 前一章末尾
9. 最近章节摘要
10. 活跃伏笔
11. 活跃冲突
12. 输出路径：`{book_dir}/chapters/chapter-{N:04d}.md`

**注意**：不输出中间文件，context 直接在上下文中传递给 Step 3。

### Step 3: 写正文

**目的**：基于 Step 2 组装的 context 创作正文。

**操作**：
1. 基于 context 创作 3000-5000 字正文
2. Write `{book_dir}/chapters/chapter-{N:04d}.md`

**约束**：
- 不读任何文件，只基于 Step 2 的 context
- 不在正文中混入英文单词
- 禁用词零触发
- 面板数据用中文格式（`键：值`，`|` 分隔）
- 章末必须留钩子

### Step 4: 摘要提取

**目的**：从正文中提取结构化信息，供后续结算使用。

**操作**：Read 刚写的正文，提取以下结构化 JSON（不写文件，直接在上下文中传递）：

```json
{
  "title": "章节标题",
  "word_count": 3500,
  "summary": "50-100字章节摘要",
  "key_events": ["事件1", "事件2", "事件3", "事件4", "事件5"],
  "character_changes": ["变化1", "变化2", "变化3", "变化4"],
  "foreshadowing_activity": ["伏笔活动1", "伏笔活动2", "伏笔活动3", "伏笔活动4"],
  "conflicts": ["冲突1", "冲突2", "冲突3"]
}
```

**字段约束**：key_events ≤5，character_changes ≤4，foreshadowing_activity ≤4，conflicts ≤3。

### Step 5: 角色状态追踪

**目的**：识别角色实质性状态变化，更新 character-state.json。

**操作**：
1. Read `{book_dir}/character-state.json`
2. Read `{book_dir}/角色.md`
3. 基于正文内容，判断哪些角色有实质性状态变化
4. 只记录关系/实力/知晓秘密/立场等变化，被顺带提到不算
5. Write `{book_dir}/character-state.json`（覆盖写入）

**更新格式**：
```json
{
  "角色名": {
    "last_seen": N,
    "changes": [
      "第X章：变化描述",
      "第N章：本次变化"
    ]
  }
}
```

### Step 6: 结算

**目的**：基于 Step 4 的 JSON 更新所有台账文件。

#### 6.1 更新 chapter-index.json

1. Read `{book_dir}/chapter-index.json`
2. 去重：如果已存在 number=N 的条目，删除旧条目
3. 追加新条目（字段见 `references/data-schemas.md`）
4. 摘要瘦身：
   - 距当前章 >5：只保留 number + title，详情追加到 `chapters/archive-summary.md`
   - 距当前章 3-5：只保留 number + title + summary
   - 距当前章 ≤2：保留完整字段
   - 总大小 ≤5000 字节
5. Write `{book_dir}/chapter-index.json`

#### 6.2 更新伏笔台账.json

1. Read `{book_dir}/伏笔台账.json`
2. 获取已有最大 ID：用 Bash 执行 `grep -oP 'f\d+' {file} | sort -u | tail -1` 或手动扫描
3. 遍历 Step 4 的 foreshadowing_activity 数组：
   - 语义去重：判断是否与已有条目语义重复（措辞不同但含义相同）
   - 如不重复，分配新 ID：f{max_id + 1:03d}
   - 追加条目：`{"id": "fXXX", "content": "内容", "planted_chapter": N, "status": "活跃"}`
4. Write `{book_dir}/伏笔台账.json`

#### 6.3 更新冲突台账.json

1. Read `{book_dir}/冲突台账.json`
2. 获取已有最大 ID
3. 遍历 Step 4 的 conflicts 数组：
   - 语义去重
   - 如不重复，分配新 ID：c{max_id + 1:03d}
   - 追加条目：`{"id": "cXXX", "content": "内容", "introduced_chapter": N, "status": "活跃"}`
4. Write `{book_dir}/冲突台账.json`

#### 6.4 更新 session-handoff.md

1. Read 所有台账获取活跃条目
2. Read chapter-index.json 获取最近 3 章概要
3. Read character-state.json 获取角色状态
4. 按模板（见 `references/data-schemas.md`）生成 handoff 文本
5. Write `{book_dir}/session-handoff.md`

#### 6.5 台账瘦身（每 5 章执行一次）

当 N 为 5 的倍数时：
1. 遍历伏笔台账：status=已兑现 → 删除 note；status=已推进且距 planted_chapter>10 → 删除 note；其他活跃条目 → note 截断到 80 字
2. 对冲突台账执行相同操作

### Step 7: 审计

**目的**：12 项质量检查。

**操作**：
1. Read 正文
2. Read 前一章正文（如存在）
3. Read 设定文件（世界观、角色）
4. Read 台账文件
5. Read 禁用词列表
6. 逐项检查（完整清单见 `references/audit-checklist.md`）
7. Write `{book_dir}/chapters/chapter-{N:04d}-audit.md`

**输出格式**：
```
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
```

### 审计后可选：逻辑审查

审计通过后，如果用户要求或章节 >3，执行逻辑审查：
- Read 世界观、角色、大纲、chapter-index.json、本章正文、前 1-2 章正文
- 4 类检查（见 `references/audit-checklist.md`）
- 跨章验证：对角色名、异能名称、数字/编号、时间表述做 Grep 全文搜索
- 发现 🔴 问题 → 必须修复后才能提交
- 发现 🟡 问题 → 展示给用户决定

## 用户确认点（不可跳过）

| 场景 | 操作 |
|------|------|
| 审计通过 | 展示结论+字数+摘要，问"确认发布？" |
| 需小修 | 展示问题，问"自动修复还是手动改？" |
| 需重写 | 展示原因，问"重写还是调大纲？" |

## 章节修改与同步

任何导致正文内容变化的操作，完成后必须执行 resync（详见 `references/sync-protocol.md`）：

1. Read 修改后的正文 → 提取结构化 JSON
2. 更新 chapter-index + 台账 + handoff + 角色状态
3. 删除旧审计报告
4. 大段重写（>30% 内容变动）→ 重新审计；小幅修补 → 跳过

## session-handoff.md 规则

必须列出**全部**活跃伏笔和冲突（status≠已兑现/已关闭），不做选择性删减。最近 3 章概要从 archive-summary.md 取。

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
| constraint_extractor 未生成文件 | 确认 Step 1 已执行并写入 constraints 文件后再进入 Step 2 |
| 台账 JSON 损坏 | 重新 Read 正文执行 Step 4 提取 |
| 摘要/事件提取缺字段 | 用默认值补全后传给 Step 6 |
| 台账去重遗漏 | Step 6 后执行台账健康检查（见 `references/sync-protocol.md`） |
| 世界观过大淹没关键规则 | 在 context 顶部用 `## 核心约束（不可违反）` 单独提取关键规则 |
| 大纲与设定矛盾 | 两者同时呈现，让 writer 自行判断 |

## 参考文件索引

| 文件 | 用途 |
|------|------|
| `references/data-schemas.md` | 所有 JSON 数据结构定义 |
| `references/context-budget.md` | 上下文裁剪策略 |
| `references/audit-checklist.md` | 12 项审计 + 4 类逻辑审查完整清单 |
| `references/sync-protocol.md` | resync 协议 + 台账健康检查 |
| `shared/writing-rules.md` | 默认写法规则 |
| `shared/banned-words.md` | 禁用词列表 |
