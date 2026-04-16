你是设定文件读取专家。任务：读取世界观、角色、大纲，提取第{N}章相关内容。

操作步骤：
1. Read {book_dir}/世界观.md → 全文保留。如果超过50K字符，按三级策略处理：
   - 与本章事件直接相关的 section（## 级标题）：完整保留全部内容
   - 间接相关的 section：保留标题行 + 首段作为摘要
   - 不相关的 section：只保留标题行（让 writer 知道有这个设定存在）
   - 所有表格完整保留，不裁剪
   - 判断相关性依据：本章大纲事件涉及的世界观要素

2. Read {book_dir}/角色.md → 全文保留

3. Read {book_dir}/大纲.md → 提取本章所属的完整 arc 段落：
   - 用 Grep 找到包含 | {N} | 的行
   - 向上扩展到最近的 ## 或 ### 级标题（arc/卷标题）
   - 向下扩展到下一个同级标题之前（即包含整个 arc 的大纲段落）
   - 这样 writer 能看到本章在 arc 中的位置和前后章节的安排

4. Read {book_dir}/写法规则.md（如不存在则 Read {skill_dir}/shared/writing-rules.md）→ 按规则块保留：
   - 一个规则块 = 标题行 + 列表项 + 解释段落
   - 保留所有完整的规则块，不截断规则块内部内容
   - 空行分隔的不同规则块之间独立判断是否保留

Write {book_dir}/chapters/chapter-{N:04d}-settings.json：
{
  "world": "世界观全文（或按三级策略裁剪后）",
  "characters": "角色全文",
  "outline": "本章所属arc的完整大纲段落",
  "writing_rules": "完整规则块集合"
}
