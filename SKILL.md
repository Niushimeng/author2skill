---
name: author2skill
description: |
  从一位小说作家的作品样本中蒸馏写作风格档案，生成可复用的仿写 skill。
  触发词：蒸馏风格 / 风格蒸馏 / 分析作者风格 / 做一个仿写 skill / author style distillation / "把 XX 的风格做成 skill"。
  不适用于：方法论拆解（用 book2skill）、书摘/读后感、简单的人物分析。
---

# author2skill — 从作品中蒸馏作者写作风格的元 skill

## 使命

从小说文本样本中抽取一位作家的**文体 DNA**，生成一份结构化的风格档案（STYLE_PROFILE.md），再将其封装成一个可被 agent 直接调用的仿写 skill（格式见 `templates/SKILL.template.md`）。

**边界**:
- ✅ 做: 从虚构作品中提取文体特征、句法节奏、叙事手法、对话风格、母题、签名句式、反模式
- ❌ 不做: 方法论拆解（那是 book2skill）、书摘/读后感、角色扮演（那是 nuwa-skill）
- ❌ 不做: 在没有文本样本的情况下"凭印象"蒸馏风格

## 与 book2skill / nuwa-skill 的生态定位

| 维度 | book2skill | author2skill（本 skill） | nuwa-skill |
|---|---|---|---|
| 蒸馏对象 | 非虚构书籍的方法论 | 小说作者的文体风格 | 人的思维方式 |
| 提取维度 | 框架 / 原则 / 清单 / 案例 | 句法 / 词汇 / 叙事 / 对话 / 母题 / 签名 | 表达 DNA / 思维模型 |
| 输出物 | 多个原子化 skill（每个含 RIA++） | 一个风格档案 + 一个仿写 skill | 人设 profile |
| 产出格式 | SKILL.md × N（含 test-prompts.json） | STYLE_PROFILE.md + 仿写 SKILL.md × 1 | profile 文档 |
| 调用时机 | 用户遇到具体决策/问题时 | 用户想模仿某位作者写东西时 | 用户想模拟某人表达时 |

**核心区别**: book2skill 的单位是"方法论"（可拆成多个原子），author2skill 的单位是"风格"（不可拆——风格是一个整体指纹）。因此 author2skill 只产出**一个**风格 skill，而非多个。

## 核心方法论: SDP（Style Distillation Pipeline）

一个五阶段的蒸馏流水线。

```
阶段 0: 样本采集与初读     → SAMPLES.md + 初读印象
阶段 1: 六维并行提取       → candidates/ (6 类特征池)
阶段 2: 交叉验证与去噪     → 通过的稳定特征
阶段 3: 风格档案合成       → STYLE_PROFILE.md
阶段 4: 仿写 Skill 构造    → 生成 SKILL.md（可直接作为独立 skill 使用）
```

详见 `methodology/00-overview.md`。

## 输入要求

在开始前**必须**从用户处确认:

1. **作者名 + 代表作**: 用于命名和定位
2. **文本样本来源**: 至少 2-3 部作品的文本（PDF / EPUB / TXT / 粘贴文本）。样本越多，提取越准确，但至少需要**同一部作品的 3 个完整章节**作为最低门槛
3. **样本质量要求**:
   - 必须包含**叙述段落**（非对话）——用于提取叙事风格
   - 必须包含**对话场景**——用于提取对话风格
   - 必须包含**至少一个情感高潮段落**——用于提取情感表达方式
   - 最好包含**开头和结尾**——用于提取结构模式
4. **目标用途**: 仿写什么类型的故事？（影响 SKILL.md 中的触发条件设计）
5. **是否需要人工调校**: 建议首次使用时在阶段 3 后暂停，让用户对"标志性维度"手动加固

## 输出结构

```
authors/<author-slug>/
├── SAMPLES.md                    # 阶段 0 产出: 样本清单 + 初读印象
├── STYLE_PROFILE.md              # 阶段 3 产出: 完整风格档案
├── candidates/                   # 阶段 1 产出: 原始候选特征池（审计用）
│   ├── prosodic.md
│   ├── lexical.md
│   ├── narrative.md
│   ├── dialogic.md
│   ├── thematic.md
│   └── signature.md
├── verified/                     # 阶段 2 产出: 验证通过的特征
│   └── verified-features.md
└── SKILL.md                      # 阶段 4 产出: 可调用的仿写 skill
```

## 执行流程 (严格按顺序)

### 阶段 0 — 样本采集与初读

1. 从用户提供的文本中，按 `methodology/01-stage0-sampling.md` 的规则选取**代表段落**
2. 对每个段落做初读标注：这段体现了作者的什么特点？
3. 按 `templates/SAMPLES.md.template` 填充，写入 `authors/<slug>/SAMPLES.md`
4. 展示初读印象给用户："我初步感知到这些风格特征，方向对吗？" 得到确认再进入阶段 1

### 阶段 1 — 六维并行提取

**并行** spawn 6 个 sub-agent（使用 Agent 工具，一次调用中发起 6 个）:

| sub-agent | 读取的 prompt | 提取维度 | 产出 |
|---|---|---|---|
| 韵律分析师 | `extractors/prosodic-extractor.md` | 句长、节奏、段落模式、留白 | `candidates/prosodic.md` |
| 词汇分析师 | `extractors/lexical-extractor.md` | 词频、词层、偏好/回避、方言标记 | `candidates/lexical.md` |
| 叙事分析师 | `extractors/narrative-extractor.md` | 视角、时态、信息释放、场景转换 | `candidates/narrative.md` |
| 对话分析师 | `extractors/dialogic-extractor.md` | 对话占比、归因风格、角色声音分化 | `candidates/dialogic.md` |
| 母题分析师 | `extractors/thematic-extractor.md` | 母题、哲学体系、情感色板、意象系统 | `candidates/thematic.md` |
| 签名分析师 | `extractors/signature-extractor.md` | 签名句式、标志性场景、反模式、独特手法 | `candidates/signature.md` |

每个 sub-agent 独立读样本、独立提取、独立输出到 `authors/<slug>/candidates/`。

### 阶段 2 — 交叉验证与去噪

对每个候选特征执行三重验证（详见 `methodology/03-stage2-verify.md`）:

- **V1 稳定性**: 在 ≥2 个独立段落/章节中出现？（偶现的不算风格）
- **V2 排除干扰**: 是作者的选择，还是题材/角色的需要？（排除题材约束导致的假特征）
- **V3 可操作性**: 能转化为仿写指令吗？（"文笔优美"不可操作，"平均句长 15 字，短句用于转折"可操作）

通过的特征写入 `verified/verified-features.md`。不通过的保留原因——允许用户事后捞回。

### 阶段 3 — 风格档案合成

1. 将通过验证的六维特征按 `templates/STYLE_PROFILE.md.template` 合成完整风格档案
2. 为每个维度提炼**可执行的仿写指令**（不只是描述，而是"要怎么做"）
3. 标注**反模式**（"绝对不要做的事"——这往往比"要做的事"更能抓住风格）
4. **暂停，展示给用户**: "这是风格档案，特别是标★的签名维度，请确认或调整"
5. 用户确认后进入阶段 4

### 阶段 4 — 仿写 Skill 构造

1. 将 STYLE_PROFILE.md 转化为一个独立的 SKILL.md，格式参照 `templates/SKILL.template.md`
2. SKILL.md 包含:
   - `name`: `<author-slug>`
   - `description`: 触发条件 + 何时不用
   - 核心思想体系（从母题维度提取）
   - 写作风格特点（从六维档案浓缩）
   - 典型写作范式（从签名维度提取，含示例段落）
   - 写作注意事项（从反模式提取）
3. 写入 `authors/<slug>/SKILL.md`
4. 该 SKILL.md 可以独立复制到 `.claude/skills/` 下直接使用

## 质量红线 (违反则阻止输出)

1. **最低样本量**: 至少 3 个完整章节，且必须覆盖叙述 + 对话 + 情感段落
2. **可操作性**: 每条风格特征必须转化为具体指令（"用短句写转折"而非"节奏感好"）
3. **反模式**: SKILL.md 必须包含至少 3 条反模式（"不要做什么"）
4. **签名段落**: SKILL.md 必须包含至少 2 段作者风格的示例文本（从原文摘取，≤200 字/段）
5. **用户确认**: 阶段 0 和阶段 3 必须暂停等用户确认，不可静默跳过

## 调用惯例

- **永远先确认样本** — 没有足够文本就停下来问
- **阶段之间主动汇报进度** — 不要静默跑完再 dump 结果
- **保留审计轨迹** — candidates/ 和 verified/ 都要留
- **建议人工调校** — 特别是"签名维度"，自动生成的泛化性好但可能缺少灵魂
