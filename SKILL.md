---
name: author2skill
description: |
  从一位小说作家的作品样本中蒸馏写作风格档案，生成可复用的仿写 skill。
  触发词：蒸馏风格 / 风格蒸馏 / 分析作者风格 / 做一个仿写 skill / author style distillation / distill author style / "把 XX 的风格做成 skill"。
  不适用于：方法论拆解（用 book2skill）、书摘/读后感、简单的人物分析、无样本的"凭印象"描述。
---

# author2skill — 从作品中蒸馏作者写作风格的元 skill

## 使命

从小说文本样本中抽取一位作家的**文体 DNA**，生成结构化的风格档案（STYLE_PROFILE.md），再封装成可被 agent 直接调用的仿写 skill（格式见 `templates/SKILL.template.md`）。

**边界**：

- ✅ 做：从虚构作品中提取文体特征、句法节奏、叙事手法、对话风格、母题、签名句式、反模式
- ❌ 不做：方法论拆解（→ book2skill）、书摘/读后感、角色扮演（→ nuwa-skill）、无文本样本的"凭印象"蒸馏

## 与 book2skill / nuwa-skill 的生态定位

| 维度 | book2skill | author2skill | nuwa-skill |
|---|---|---|---|
| 蒸馏对象 | 非虚构方法论 | 小说作者文体 | 人的思维方式 |
| 提取维度 | 框架/原则/清单/案例 | 句法/词汇/叙事/对话/母题/签名 | 表达 DNA/思维模型 |
| 输出物 | 多个原子化 skill | 一个风格档案 + 一个仿写 skill | 人设 profile |
| 调用时机 | 用户遇到具体决策 | 用户想模仿某位作者写东西 | 用户想模拟某人表达 |

**核心区别**：方法论可拆成多个原子；风格不可拆——风格是整体指纹。author2skill 只产出**一个**风格 skill。

## 核心方法论：SDP（Style Distillation Pipeline）

```
阶段 0：样本采集与初读     → SAMPLES.md
阶段 1：六维并行提取       → candidates/ (6 类特征池)
阶段 2：交叉验证与去噪     → verified-features.md
阶段 3：风格档案合成       → STYLE_PROFILE.md
阶段 4：仿写 Skill 构造    → SKILL.md（可直接调用）
```

详见 `methodology/00-overview.md`。

## 输入要求（启动前必须满足；任一缺失则不得进入阶段 0）

1. **作者名 + 代表作**：用于命名 author-slug
2. **文本样本**：至少 2-3 部作品。**硬门槛：同一部作品的 3 个完整章节**
3. **样本质量（必须全部满足）**：含叙述段落 + 含对话场景 + 含 ≥1 情感高潮段落 + 含开头和结尾
4. **目标用途**：仿写什么类型的故事？
5. **是否需要人工调校**：首次使用者必须在阶段 3 后暂停人工调校（非可选）

## 输出结构

```
authors/<author-slug>/
├── SAMPLES.md                    # 阶段 0
├── STYLE_PROFILE.md              # 阶段 3
├── candidates/                   # 阶段 1（审计用）
│   ├── prosodic.md  lexical.md  narrative.md
│   ├── dialogic.md  thematic.md  signature.md
├── verified/
│   ├── verified-features.md      # 阶段 2
│   └── rejected.md               # 阶段 2 淘汰归档
└── SKILL.md                      # 阶段 4
```

## 执行流程（严格按顺序）

### 阶段 0 — 样本采集与初读

1. 按 `methodology/01-stage0-sampling.md` 选取代表段落。**每类（叙述/对话/情感）至少 5 段，单段 200-800 字**
2. 对每段做初读标注
3. 按 `templates/SAMPLES.md.template` 填充，写入 `authors/<slug>/SAMPLES.md`
4. 展示初读印象给用户

🔴 **CHECKPOINT · 🛑 STOP**：必须暂停等用户确认"方向对"再进入阶段 1。用户回答"不对"→ 重选段，不得继续。

### 阶段 1 — 六维并行提取

**并行** spawn 6 个 sub-agent（一次调用发起 6 个）：

| sub-agent | 提取维度 | 产出 |
|---|---|---|
| 韵律分析师 (`extractors/prosodic-extractor.md`) | 句长、节奏、段落、留白 | `candidates/prosodic.md` |
| 词汇分析师 (`extractors/lexical-extractor.md`) | 词频、词层、偏好/回避、方言 | `candidates/lexical.md` |
| 叙事分析师 (`extractors/narrative-extractor.md`) | 视角、时态、信息释放、场景转换 | `candidates/narrative.md` |
| 对话分析师 (`extractors/dialogic-extractor.md`) | 对话占比、归因风格、声音分化 | `candidates/dialogic.md` |
| 母题分析师 (`extractors/thematic-extractor.md`) | 母题、哲学、情感色板、意象 | `candidates/thematic.md` |
| 签名分析师 (`extractors/signature-extractor.md`) | 签名句式、标志场景、反模式 | `candidates/signature.md` |

🔴 **CHECKPOINT**：6 个文件全部生成后展示清单（每文件特征条数）。任一文件为空或 < 3 条 → 进入失败表 F2。

### 阶段 2 — 交叉验证与去噪

三重验证（详见 `methodology/03-stage2-verify.md`）：

- **V1 稳定性**：≥2 个独立段落/章节出现？
- **V2 排除干扰**：作者选择 vs 题材约束？
- **V3 可操作性**：能转化为仿写指令？

通过→`verified/verified-features.md`；未通过→`verified/rejected.md`（归档不删除，供事后捞回）。

🔴 **CHECKPOINT**：通过特征数 < 10 → 进入失败表 F3。

### 阶段 3 — 风格档案合成

1. 按 `templates/STYLE_PROFILE.md.template` 合成完整档案
2. 每个维度提炼**可执行的仿写指令**（含具体动作动词，非描述）
3. 标注**反模式**（≥3 条，"绝对不要做的事"）

🔴 **CHECKPOINT · 🛑 STOP**：展示档案给用户，**标★的签名维度必须由用户确认或调整**。未明确确认前不得进入阶段 4。

### 阶段 4 — 仿写 Skill 构造

1. 按 `templates/SKILL.template.md` 生成 SKILL.md，必须含：
   - `name`：`<author-slug>`
   - `description`：触发条件 + 何时不用 + 触发词清单
   - 核心思想（母题维度）
   - 写作风格特点（六维档案浓缩）
   - 典型范式（签名维度 + 示例段落）
   - 反模式（≥3 条）
2. 写入 `authors/<slug>/SKILL.md`，可独立复制到 runtime 的 skills 目录调用

🔴 **CHECKPOINT**：交付前对照"质量红线"5 条逐一勾选，任一不满足 → 回对应阶段补救。

## 失败模式与 fallback（if-then 三段式）

| ID | 触发条件 | 一线修复 | 仍失败兜底 |
|----|---------|---------|----------|
| F1 | 用户文本 < 3 个完整章节 | 告知门槛 + 列缺失 + 询问能否补齐 | 终止流程，输出"样本不足"诊断；**不得**降级运行 |
| F2 | 阶段 1 某 candidates/*.md 为空或 < 3 条 | 重跑该 sub-agent（最多 2 次），prompt 显式追加缺失维度示例 | 标注该维度"证据不足"，STYLE_PROFILE.md 显式 N/A；**不得**自由发挥 |
| F3 | 阶段 2 通过特征 < 10 | 放宽 V1 阈值（≥2→≥1，且必须在情感高潮段出现） | 暂停告知"风格信号弱"，征求是否追加样本 |
| F4 | 阶段 3 用户不认可档案 | 询问具体不认可维度 → 回 rejected.md 捞回相关特征重评 | 第 2 次仍不认可 → 暂停，建议人工接管 |
| F5 | 提供的不是虚构作品 | 终止 + 重定向到 book2skill | 用户坚持 → 明确拒答"author2skill 不适用" |
| F6 | 有作者名但无样本（"我想要余华风格"） | 拒绝启动 + 告知必须有文本 + 提供获取建议 | 用户拒补 → 终止 |
| F7 | 阶段 1 sub-agent 失败/超时 | 重试 1 次；仍失败 → 串行降级跑 6 个，标 `serial_fallback` | 串行也失败 → 输出已完成维度 + 失败清单 |
| F8 | 生成的 SKILL.md > 模板 200% 体积 | 拒绝输出，回阶段 4 精简 | 仍超 → 拆主 SKILL.md + REFERENCE.md |

**原则**：异常先告知用户，按上表处理；**绝不静默跳过或静默失败**。

## 质量红线（违反则阻止输出）

1. **最低样本量**：≥3 个完整章节，覆盖叙述 + 对话 + 情感段落
2. **可操作性**：每条特征必须转化为具体指令（"用短句写转折"而非"节奏感好"）
3. **反模式**：SKILL.md 至少 3 条反模式
4. **签名段落**：SKILL.md 至少 2 段原文示例（≤200 字/段）
5. **用户确认**：阶段 0/3 必须暂停等确认，不可静默跳过

## 反模式黑名单（蒸馏过程绝对不要做）

| # | 反模式 | 替代做法 |
|---|--------|---------|
| 1 | **凭印象蒸馏不读样本**（输出会被刻板印象污染） | 每条特征必须能引述原文段落 |
| 2 | **跨题材混合样本**（得到平均值，谁都不像） | 跨题材时分别提取，由用户选保留哪个子风格 |
| 3 | **跳过反模式提取**（"不用什么"才是风格指纹） | 阶段 3 必须输出 ≥3 条反模式 |
| 4 | **把题材当风格**（"写农村"是题材不是文体 DNA） | V2 必须排除题材约束，只保留主动语言选择 |
| 5 | **用模糊形容词代替可操作指令**（仿写无法落地） | V3 强制，每条含可量化参数或具体动作 |
| 6 | **静默跳过用户确认**（后续基于错误前提推进） | 检查点必须显式暂停，用户不回复就不推进 |
| 7 | **样本太少硬撑**（< 3 章硬跑 → 提取的是噪声） | 触发 F1，不得降级运行 |
| 8 | **SKILL.md 加戏脱离档案**（破坏审计链） | SKILL.md 必须可追溯到 STYLE_PROFILE.md 对应条目 |

**使用方式**：阶段 0/1/3/4 开始前对照一次，命中任一条 → 改方案重做。

## 调用惯例

- **必须先确认样本** — 不满足 3 章门槛立刻触发 F1
- **阶段间必须汇报进度** — 不得静默跑完再 dump
- **必须保留审计轨迹** — `candidates/`、`verified/`、`verified/rejected.md` 全部保留
- **必须人工调校签名维度** — 自动生成泛化好但缺灵魂，签名由用户最终确认
