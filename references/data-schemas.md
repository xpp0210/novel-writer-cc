# 数据 Schema 定义

本文档定义所有 JSON 数据结构的标准格式。所有台账文件必须严格遵循此格式。

## chapter-index.json

```json
[
  {
    "number": 1,
    "title": "章节标题",
    "status": "已通过",
    "word_count": 3500,
    "summary": "50-100字章节摘要",
    "key_events": ["事件1", "事件2", "事件3", "事件4", "事件5"],
    "character_state_changes": ["变化1", "变化2", "变化3", "变化4"],
    "foreshadowing_activity": ["伏笔活动1", "伏笔活动2", "伏笔活动3", "伏笔活动4"]
  }
]
```

**字段约束**：
- `number`：整数，章节编号
- `status`：字符串，枚举值 `已通过` / `待审计` / `需修改`
- `key_events`：最多5条，简短描述
- `character_state_changes`：最多4条
- `foreshadowing_activity`：最多4条

**瘦身规则**（按字段分层，不设字节上限）：
- 距当前章 >5：保留 `number` + `title` + `key_events`（key_events 是核心骨架，永不丢弃），详情归档到 `chapters/archive-summary.md`
- 距当前章 3-5：保留 `number` + `title` + `summary` + `key_events`
- 距当前章 ≤2：保留完整字段

## 伏笔台账.json

```json
[
  {
    "id": "f001",
    "content": "伏笔内容描述",
    "planted_chapter": 3,
    "status": "活跃",
    "expected_payoff": 8,
    "note": "补充说明（可选）"
  }
]
```

**字段约束**：
- `id`：字符串，格式 `f` + 3位数字（如 f001、f002）
- `planted_chapter`：整数，埋设章节号
- `status`：字符串，枚举值 `活跃` / `已推进` / `已兑现` / `已关闭`
- `expected_payoff`：整数（可选），预计兑现章节
- `note`：字符串（可选），补充说明

**ID 分配**：读取台账中所有 `f\d+` 格式的 ID，取最大值 +1。

## 冲突台账.json

```json
[
  {
    "id": "c001",
    "content": "冲突内容描述",
    "introduced_chapter": 2,
    "status": "活跃",
    "expected_resolution": 10,
    "note": "补充说明（可选）"
  }
]
```

**字段约束**：
- `id`：字符串，格式 `c` + 3位数字（如 c001、c002）
- `introduced_chapter`：整数，引入章节号
- `status`：字符串，枚举值 `活跃` / `升级` / `已解决` / `冻结`
- `expected_resolution`：整数（可选），预计解决章节

## character-state.json

```json
{
  "角色名": {
    "last_seen": 5,
    "changes": [
      "第3章：觉醒F级异能",
      "第5章：得知导师真实身份"
    ]
  }
}
```

**字段约束**：
- `last_seen`：整数，最后出场章节号
- `changes`：字符串数组，每条格式 `第X章：变化描述`

**更新规则**：只记录实质性状态变化（关系、实力、知晓秘密、立场转变等），被顺带提到不算。

## session-handoff.md

```markdown
# Session Handoff

## 当前进度
第N章已完成，下一章编号：N+1
标题：章节标题

## 最近3章概要
- 第X章「标题」摘要内容
- 第Y章「标题」摘要内容
- 第Z章「标题」摘要内容

## 角色当前状态
- 角色名(第X章): 变化1; 变化2

## 关键待处理伏笔
- f001(伏笔内容)
- f002(伏笔内容)

## 关键待处理冲突
- c001(冲突内容)

## 台账状态
伏笔X条活跃 / 冲突Y条活跃
```

**同步规则**：
- 伏笔条数必须与 `伏笔台账.json` 中 status≠已兑现/已关闭 的条数一致
- 冲突条数必须与 `冲突台账.json` 中 status≠已解决 的条数一致
- 概要从 `chapter-index.json` 的 `key_events` 同步，用分号分隔
- 概要无重复前缀（如"第3章「第3章「标题」」"是 bug）

## chapter-{N:04d}-extract.json（extractor 子代理输出）

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

**字段约束**：
- `title`：字符串，章节标题
- `word_count`：整数，正文字数
- `summary`：字符串，50-100字
- `key_events`：最多5条
- `character_changes`：最多4条
- `foreshadowing_activity`：最多4条
- `conflicts`：最多3条

**用途**：settler 子代理读取此文件，更新所有台账。
