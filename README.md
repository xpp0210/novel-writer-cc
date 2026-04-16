# Novel Writer CC

基于 Claude Code 的 AI 小说写作引擎，采用子代理（Sub-Agent）架构，覆盖从灵感到成书的完整 Pipeline，专为中文长篇网文优化。

## 特性

- **子代理架构**：主 Session 只做编排，14 个独立子代理通过文件通信协作完成写作
- **并行调度**：无依赖步骤自动并行（上下文读取 3 路并行、状态结算 4 路并行、审计 2 路并行）
- **语义完整性裁剪**：按结构边界（场景、规则块、完整句子）裁剪上下文，禁止字符数截断
- **全链路审计**：12 项必检 + 4 类逻辑审查，跨章 Grep 验证防遗漏
- **台账驱动**：伏笔台账、冲突台账、角色状态自动维护，跨章节一致性有保障
- **断点续传**：中间文件自动检测，中断后从断点恢复
- **反 AI 痕迹**：禁用词列表 + 写法规则，消除 LLM 高频输出模式

## 架构

```
主 Session（编排器）
  │
  ├─ Step 1:  constraint_extractor     → 硬约束提取
  │
  ├─ Step 2a+2b+2c:  并行              → 设定读取 / 台账摘要 / 近文读取
  │     └─ Step 2d:  context_assembler  → 上下文组装
  │
  ├─ Step 3:  writer                   → 章节创作
  │
  ├─ Step 4:  extractor                → 结构化信息提取
  │
  ├─ Step 5+6a+6b+6c: 并行            → 角色状态 / 索引 / 伏笔 / 冲突更新
  │     └─ Step 6d:  handoff_builder   → 会话交接文档重建
  │
  └─ Step 7+8: 并行                    → 审计 / 逻辑审查
```

## 子 Skill

| Skill | 功能 | 触发词 |
|-------|------|--------|
| `novel-chapter-polish` | 章节润色，提升完读率 | 润色章节、打磨开篇 |
| `novel-logic-review` | 全量逻辑审查（4 类问题） | 检查逻辑、审查硬伤 |
| `novel-setting-audit` | 设定文件交叉审查（7 维度） | 检查设定、设定一致性 |
| `novel-setting-consistency-check` | 设定修改后全量交叉检查 | 检查一致性 |

## 安装

将本仓库克隆到 Claude Code skills 目录：

```bash
git clone https://github.com/xpp0210/novel-writer-cc.git ~/.claude/skills/novel-writer
```

## 使用

安装后重启 Claude Code，以下指令自动触发：

| 指令 | 执行内容 |
|------|---------|
| "继续写" / "写下一章" / "写第N章" | 完整写作 Pipeline |
| "创建新小说" | 初始化书籍目录和模板 |
| "审计第N章" | 单章审计 |
| "检查逻辑" | 全量逻辑审查 |
| "检查设定" | 设定文件交叉审查 |
| "润色第N章" | 章节润色 |
| "检查一致性" | 设定修改后交叉检查 |
| "修改第N章" | 修改 + 自动 resync |
| "全文重写" | 归档旧章 + 重置 + 重新开始 |

## 书籍目录结构

```
<书名>/
├── 世界观.md / 角色.md / 大纲.md / 写法规则.md
├── 伏笔台账.json / 冲突台账.json / chapter-index.json / character-state.json
├── session-handoff.md
└── chapters/
    ├── chapter-NNNN.md              # 正文
    ├── chapter-NNNN-constraints.md  # 硬约束
    ├── chapter-NNNN-settings.json  # 设定文件
    ├── chapter-NNNN-ledgers.json   # 台账摘要
    ├── chapter-NNNN-recent.json    # 近文参考
    ├── chapter-NNNN-context.json   # 最终上下文
    ├── chapter-NNNN-extract.json    # 提取 JSON
    └── chapter-NNNN-audit.md        # 审计报告
```

## 项目结构

```
novel-writer/
├── SKILL.md                          # 主 skill 定义
├── agents/                           # 子代理指令（14 个独立文件）
│   ├── 01-constraint-extractor.md
│   ├── 02a-settings-reader.md
│   ├── 02b-ledgers-reader.md
│   ├── 02c-recent-reader.md
│   ├── 02d-context-assembler.md
│   ├── 03-writer.md
│   ├── 04-extractor.md
│   ├── 05-state-tracker.md
│   ├── 06a-index-updater.md
│   ├── 06b-foreshadow-updater.md
│   ├── 06c-conflict-updater.md
│   ├── 06d-handoff-builder.md
│   ├── 07-auditor.md
│   └── 08-logic-reviewer.md
├── references/                       # 参考文档
│   ├── data-schemas.md               # JSON 数据结构定义
│   ├── context-budget.md             # 上下文裁剪策略
│   ├── audit-checklist.md            # 审计检查清单
│   └── sync-protocol.md              # resync 协议
├── shared/                           # 共享资源
│   ├── writing-rules.md              # 默认写法规则
│   └── banned-words.md               # 禁用词列表
├── novel-chapter-polish/SKILL.md     # 章节润色子 skill
├── novel-logic-review/SKILL.md       # 逻辑审查子 skill
├── novel-setting-audit/SKILL.md      # 设定审查子 skill
└── novel-setting-consistency-check/SKILL.md  # 一致性检查子 skill
```

## 依赖

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)（需支持 Agent 工具）
