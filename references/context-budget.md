# 上下文预算分配策略

## 预算总览

| 数据类型 | 预算 | 裁剪策略 |
|---------|------|---------|
| 硬约束（constraints） | ≤500字 | Step 1 生成，语义压缩 |
| 写法规则 | ≤600字 | 保留标题行+列表行+所属解释段落 |
| 大纲（本章） | ≤1200字 | Grep 提取本章段落（前后5行+所属arc标题） |
| 台账摘要 | ≤2500字 | 按条数限制，content 截断到100字（句子完整） |
| 近文参考 | ≤4500字 | 前章 key_events + 末尾2500字，前2章各500字 |
| 世界观 | 全文 | 通常 <25K 字符 |
| 角色 | 全文 | 通常 <5K 字符 |
| 角色状态 | 按需 | 每人最近5条 changes |
| 禁用词 | 全文 | ~2KB，必须完整 |

**总估算**：~35K 字符 ≈ 17K token，远低于 200K token 限制。

## 粗裁剪策略

与原项目精确裁剪不同，纯指令版采用粗裁剪：

1. **世界观 + 角色**：全文保留。如果总字符数 >50K，按 ## 级标题（section）裁剪：保留与本章事件直接相关的 section 完整内容，不相关 section 只保留标题行（让 writer 知道有这个设定存在）。保留所有表格行
2. **写法规则**：保留以 `#`、`-`、`*`、`❌`、`✅` 开头的行及其所属的完整段落，超过 600 字按段落截断
3. **大纲**：用 Grep 提取包含 `| {N} |` 的行及前后 5 行，如果上方有 ##/### 级标题则扩展到包含该标题。超过 1200 字截断
4. **台账摘要**：伏笔 ≤10 条（content 截断到 100 字，必须句子完整），冲突 ≤6 条（同上），章节摘要最近 15 条
5. **近文参考**：先读前章 extract.json 的 key_events 作为事件概要，再取正文末尾 2500 字；倒数第二、三章各取末尾 500 字

## 上下文组装顺序

按以下优先级拼接（前面的内容 writer 更容易关注到）：

1. 本章硬约束（constraints）
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
12. 输出路径

## 关键约束

- 世界观超过安全线（50K 字符）时，用 `## 核心约束（不可违反）` 在顶部单独提取关键规则，再附全文
- 大纲与设定矛盾时，两者同时呈现，让 writer 自行判断
- 面板能力展示需标注当前处于哪一层级，允许展示到什么程度

## context.json 输出格式（composer 子代理输出）

composer 子代理将组装好的上下文写入 `chapters/chapter-{N:04d}-context.json`，供 writer 子代理读取。

```json
{
  "chapter": 5,
  "output_path": "/path/to/book/chapters/chapter-0005.md",
  "constraints": "本章硬约束文本...",
  "writing_rules": "写法规则文本...",
  "banned_words": "禁用词列表...",
  "world": "世界观全文...",
  "characters": "角色全文...",
  "character_state": {"角色名": {"last_seen": 4, "changes": ["..."]}},
  "outline": "本章大纲段落...",
  "recent_text": "前一章末尾2000字...",
  "prev_chapters": [{"num": 3, "tail": "末尾500字..."}],
  "chapter_summaries": [{"num": 1, "title": "标题", "summary": "摘要"}],
  "active_foreshadow": [{"id": "f001", "content": "伏笔内容"}],
  "active_conflicts": [{"id": "c001", "content": "冲突内容"}]
}
```

**字段约束**：
- `chapter`：整数，当前章节号
- `output_path`：字符串，正文输出绝对路径
- `constraints`：≤500字
- `writing_rules`：≤600字
- `outline`：≤1200字
- `world`：全文或裁剪后的世界观
- `characters`：全文或裁剪后的角色
- `active_foreshadow`：最多10条
- `active_conflicts`：最多6条
- `chapter_summaries`：最近15条
- `recent_text`：前一章末尾2500字
- `prev_chapters`：倒数第二、三章各末尾500字（含 events 概要）
