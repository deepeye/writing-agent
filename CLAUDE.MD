# 项目：写作 Agent（文章项目）v0.7.0

## 项目意图
本项目是一个工作流驱动的写作系统，采用 **Skills + Subagents 混合架构**。
核心特色在于**强人工干预的质检（去AI味）**与**自动化自进化机制（记忆学习循环）**。
目标不仅仅是写出高质量文章，而是通过不断复盘，越写越贴近用户的真实写作习惯。

Claude 必须严格遵守工作流调度，所有关键写作节点都应当由具体的 Subagent 承担执行。

---

## 🚨 核心架构：系统双轴

本项目运作在两条核心轴线上：生产轴与进化轴。

### 1. 生产轴（可见流程）- 14阶段协作 SOP
通过 `工作流导演` Skill 发起的 14 步标准操作程序（SOP）：从需求澄清、资料搜集、构思大纲、共情设计、到实际攥写、去 AI 味及排版输出。

### 2. 进化轴（隐形闭环）- 采样与编译系统
- **编译记忆 (Stage 0)**：每次写作前，`memory-loader` 从历史复盘中提炼用户的偏好，生成 `00_memory_packet.md` 供主力写手们参考。
- **采样归因 (Stage 13)**：每次定稿后，`edit-diff-learner` 自动比对原始初稿与最终用户定稿，将修改经验映射回 15 维风格 DSL，写入 `99_episode.md`。
- **自动化清洗 (Hooks)**：全流程结束时，自动触发 `auto_clean_hook.py` 脱离 LLM 进行正则表达式级的无痕排版（`_clean.txt`）。

---

## 🛠️ 工具集全览

### 可用 Skills（3个，语义触发）
位于 `.claude/skills/` 目录：

| Skill | 触发方式 | 职责 |
|-------|---------|------|
| `工作流导演` | 所有写作请求 | **总调度器**，通过显式调用 16 个 Subagent 完成任务 |
| `公众号文章获取` | 检测到 mp.weixin.qq.com 链接 | 提取公众号文章内容 |
| `风格建模` | "学习风格"、"提取风格" | 分析并创建风格文件 (Base + Delta 架构准备) |

### 可用 Subagents（16个，显式调用）
位于 `.claude/agents/` 目录：

| Subagent | 职责 | 产物 |
|----------|------|------|
| `memory-loader` | 记忆装载 | `00_memory_packet.md` |
| `topic-generator` | 选题生成 | 候选选题列表 |
| `topic-research` | 选题调研 | 选题评分报告 |
| `writing-clarifier` | 澄清需求 | `01_theme.md` |
| `research-expert` | 调研素材 | `02_cases.md` |
| `outline-architect` | 设计大纲 | `03_outline.md` |
| `empathy-designer` | 共情点设计 | `04_empathy_map.md` |
| `concretizer` | 具象化设计 | `05_concrete_library.md` |
| `title-designer` | 标题设计 | 标题确认 |
| `writing-executor` | 写作执行 | `draft_v1.md` |
| `editor-review` | 主编审稿 | 审稿报告 + 新版稿件 |
| `pre-publish-review` | 发布前评审 | 评审报告 |
| `toutiao-reader-test` | 读者模拟 | 四关测试报告 |
| `humanizer` | 去AI味专家 | `_humanized.md` + 50分自评表 |
| `article-illustrator` | 文章配图师 | 生成配图 |
| `edit-diff-learner` | 写作复盘学习器 | `99_episode.md` |

---

## 完整协作流程图

```
用户选择写作模式 (B)
    ↓
Stage 0: 🧠 经验装载 → 使用 memory-loader → 00_memory_packet.md
    ↓
Stage 1: 📝 需求澄清 → 使用 writing-clarifier → 01_theme.md
    ↓
Stage 2: 📚 素材调研 → 使用 research-expert → 02_cases.md
    ↓
Stage 3: 🏗️ 大纲设计 → 使用 outline-architect → 03_outline.md
    ↓
Stage 4: 💖 共情构建 → 使用 empathy-designer → 04_empathy_map.md
    ↓
Stage 5: 🧱 具象支撑 → 使用 concretizer → 05_concrete_library.md
    ↓
Stage 5.5: 💡 标题设计 → 使用 title-designer → 给定5个标题供选择
    ↓
Stage 6: ✍️ 执笔写作 → 使用 writing-executor → draft_v1.md
    ↓
Stage 7: 🧐 主编审稿 → 使用 editor-review → draft_vX.md（直到通过）
    ↓
Stage 8: 🕵️ 发布评审 → 使用 pre-publish-review
    ↓
Stage 9: 👿 读者测试 → 使用 toutiao-reader-test
    ↓
Stage 10: 🤖 强制净化 → 使用 humanizer 去AI味 → _humanized.md
    ↓
Stage 11: 🎨 可选配图 → 使用 article-illustrator 插入配图
    ↓
Stage 12: 📤 终局排版 → 由 Hook 自动生成纯净排版 .txt 文件
    ↓
Stage 13: 🔄 经验蒸馏 → 使用 edit-diff-learner 沉淀本次经验 → 99_episode.md
```

---

## ⚠️ 调用核心语法规范

**必须使用以下格式调用 Subagent**：

```
使用 [subagent-name] 子代理来 [任务描述]。
项目名称：[项目名]
[其他必要参数]
```

**必须遵守的信息传递原则：**
1. 不准在对话框长篇乱造，禁止绕过 agent 写大段文章。
2. 每一个核心步骤的结论、产物必须落地保存到物理文件 `articles/[项目名]/`。
3. 后续 Agent 的启动必须显式告知其去读取哪些既往文件（特别是 `00_memory_packet.md`）。

---

## 最终原则：不可谈判的质量红线

**去AI味 > 真实感语感 > 清晰的逻辑结构 > 快**

宁愿花十个阶段产出一篇充满泥土气息的大白话，也绝不接受一篇三段式整整齐齐、满纸"不可或缺""至关重要"的高质量 AI 八股文。

---

## 版本记录
- **v0.7.0** (2026-03-15): 架构深度进化，引入自学习闭环（Stage 0 / Stage 13）和自动化排版 Hook。
- **v0.6.3** (2026-03-01): 引入 Humanizer-zh 强质检逻辑（50 分制）与致命黑名单阻断。
- **v0.5.0** (2026-01-25): 重大架构升级，启用 10 层工作流，挂载 Skill + Subagent 矩阵分离职责。
