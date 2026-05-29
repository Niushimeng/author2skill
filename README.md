# author2skill

一个元技能（meta-skill），用于分析小说作家的文本样本，提取其写作风格 DNA，生成可复用的仿写技能。兼容 Agent Skills Standard，可在任意支持 skills 的 agent runtime（Claude Code / Codex / Cursor / OpenClaw / Hermes / Gemini CLI / OpenCode 等）调用。

属于三件套工具之一：**author2skill**（小说风格）| **book2skill**（方法论提炼）| **nuwa-skill**（思维风格建模）

## 快速开始

将本目录放置到运行环境的 skills 目录后，触发以下任一关键词即可调用：

```
蒸馏风格 / 风格蒸馏 / 分析作者风格 / 做一个仿写 skill / author style distillation
```

也支持直接调用 skill 名（语法依 runtime 而定）：

```
/author2skill
```

流程会引导你：

1. 提供作者文本样本（至少3章，覆盖叙述、对话、情感场景）
2. 六个维度并行提取风格特征
3. 交叉验证与去噪
4. 合成风格档案（需用户确认）
5. 生成独立的 `SKILL.md`，可直接调用来模仿该作者写作

## 工作原理：SDP 流水线

核心方法论为 **SDP（风格蒸馏流水线）**，共五个阶段，关键节点有人工审核。

| 阶段 | 名称 | 做什么 |
|------|------|--------|
| 0 | 样本采集 | 用户提供代表性文本，记录初步印象 |
| 1 | 并行提取 | 6个子智能体同时分析韵律、词汇、叙事、对话、主题、标志性特征 |
| 2 | 交叉验证 | 每个候选特征通过稳定性、干扰排除、可操作性三项检验 |
| 3 | 风格档案合成 | 验证后的特征合并为结构化的 `STYLE_PROFILE.md` |
| 4 | 技能构建 | 将档案转化为独立可用的 `SKILL.md` |

### 六个维度

- **韵律（Prosodic）** — 句式节奏、段落呼吸、叙事节拍
- **词汇（Lexical）** — 用词频率、语域、偏好词与回避词
- **叙事（Narrative）** — 视角、时态、信息释放、场景转换
- **对话（Dialogic）** — 对话占比、引语标注风格、角色声线区分
- **主题（Thematic）** — 母题、哲学观、情感色谱、意象体系
- **标志性（Signature）** — 标志句式、标志性场景、反模式、独特技法

## 项目结构

```
author2skill/
  SKILL.md                          # 入口（调用这个）
  methodology/                      # 流水线阶段定义
    00-overview.md                  # SDP 设计哲学
    01-stage0-sampling.md           # 样本选择规则
    02-stage1-parallel-extract.md   # 6智能体编排
    03-stage2-verify.md             # 验证标准
    04-stage3-profile.md            # 档案合成规则
    05-stage4-skill.md              # 技能构建规则
  extractors/                       # 各子智能体的提示词
  templates/                        # 输出模板
  authors/                          # 已完成的提取案例
    xugongzi/                       # 示例：徐公子胜治
      SKILL.md                      # 生成的可用技能
```

## 示例

`authors/xugongzi/` 目录包含徐公子胜治的完整提取案例（道释哲理小说，古今交融的文风）。可查看 `authors/xugongzi/SKILL.md` 了解最终产出的样子。

## 设计原则

- **风格是整体指纹** — 每位作者产出一个技能，而非零散特征的堆砌
- **可操作优先于描述性** — 每个特征都转化为具体的"这样做"或"不要这样做"
- **反模式同样重要** — 作者回避什么和使用什么一样具有定义性
- **人工审核** — 流水线在阶段0和阶段3暂停，等待用户确认
- **保留审计轨迹** — 中间的 `candidates/` 和 `verified/` 目录供回溯审查
