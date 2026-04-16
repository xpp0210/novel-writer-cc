# 上下文预算分配策略

## 核心原则：语义完整性优先

**禁止字符数截断。** 所有裁剪必须按结构边界（section、场景、规则块、完整句子）进行。
宁可多保留内容，也不要截断导致语义不完整。

## 预算总览

| 数据类型 | 策略 | 边界单位 |
|---------|------|---------|
| 硬约束（constraints） | 完整规则清单，不截断 | 每条规则（条件+约束+后果） |
| 写法规则 | 按规则块保留 | 一个规则块 = 标题+列表+解释 |
| 大纲（本章） | 提取完整 arc 段落 | 从 arc 标题到下一个同级标题 |
| 台账摘要 | content 原样或改写为一句话摘要 | 语义单元（who/what/why） |
| 近文参考 | 按完整场景保留 | 以空行分隔的场景 |
| 世界观 | 三级策略 | section（## 级标题） |
| 角色 | 全文 | 完整文件 |
| 角色状态 | 近期逐条+早期合并摘要 | 每条 change |
| 禁用词 | 全文 | 完整文件 |

## 结构边界裁剪策略

1. **世界观**：三级 section 策略
   - 与本章直接相关：完整保留 section 全部内容
   - 间接相关：保留标题行 + 首段摘要
   - 不相关：只保留标题行（让 writer 知道有这个设定）
   - 所有表格完整保留

2. **写法规则**：按规则块保留
   - 一个规则块 = 标题行 + 列表项 + 解释段落
   - 保留所有完整规则块，不截断规则块内部

3. **大纲**：提取完整 arc
   - 从本章所在行的最近 ##/### 标题到下一个同级标题
   - writer 能看到本章在 arc 中的位置

4. **台账摘要**：语义保留
   - content 原样保留（通常已简洁）
   - 过长则改写为一句话摘要（必须包含 who/what/why 三要素）
   - 禁止字符数截断

5. **近文参考**：场景边界切分
   - 前一章：最后 2 个完整场景 + key_events 概要
   - 前两/三章：最后 1 个完整场景 + events 概要
   - 场景以空行分隔，不从场景中间切开

6. **角色状态**：分层保留
   - 全部 changes 保留，不丢弃
   - 超过5条时，最早的合并为累积摘要

7. **chapter-index 瘦身**：字段分层
   - 旧章（>5）：保留 number + title + key_events（骨架永不丢）
   - 中章（3-5）：+ summary
   - 近章（≤2）：完整字段
   - 不设字节上限

## 上下文组装顺序

按以下优先级拼接（前面的内容 writer 更容易关注到）：

1. 本章硬约束（constraints）
2. 写作规则
3. 禁用词
4. 世界观
5. 角色
6. 角色当前状态
7. 本章大纲
8. 前一章末尾场景
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
  "constraints": "本章完整硬约束规则清单",
  "writing_rules": "完整规则块集合",
  "banned_words": "禁用词列表（完整）",
  "world": "世界观全文或三级策略裁剪后",
  "characters": "角色全文",
  "character_state": {"角色名": {"last_seen": 4, "changes": ["近期逐条", "早期合并摘要"]}},
  "outline": "本章所属arc的完整大纲段落",
  "recent_events": ["前一章key_events概要"],
  "recent_text": "前一章最后2个完整场景",
  "prev_chapters": [{"num": 3, "tail": "最后1个完整场景", "events": ["事件概要"]}],
  "chapter_summaries": [{"num": 1, "title": "标题", "summary": "摘要", "key_events": [...]}],
  "active_foreshadow": [{"id": "f001", "content": "完整内容或一句话摘要"}],
  "active_conflicts": [{"id": "c001", "content": "完整内容或一句话摘要"}]
}
```

**字段说明**：
- `constraints`：完整规则清单，每条包含条件+约束+后果
- `writing_rules`：完整规则块，不截断
- `outline`：完整 arc 段落，不截断
- `world`：三级 section 策略裁剪后
- `active_foreshadow`：最多10条，content 不截断
- `active_conflicts`：最多6条，content 不截断
- `chapter_summaries`：最近15条，按距离分层保留字段
- `recent_text`：前章最后2个完整场景（按空行分隔的场景边界）
- `prev_chapters`：倒数第二/三章各最后1个完整场景
