# 阶段 1: 六维并行提取

## 目标

用 6 个并行 sub-agent 从样本中提取风格特征，产出 6 个候选特征文件。

## 并行执行

使用 Agent 工具**一次调用中发起 6 个 sub-agent**，每个 sub-agent:
- 读取自己的 extractor prompt（`extractors/<name>.md`）
- 独立读取样本文本
- 独立提取特征
- 独立输出到 `authors/<slug>/candidates/<dimension>.md`

## 6 个 sub-agent 清单

| # | 名称 | Extractor Prompt | 产出文件 |
|---|---|---|---|
| 1 | 韵律分析师 | `extractors/prosodic-extractor.md` | `candidates/prosodic.md` |
| 2 | 词汇分析师 | `extractors/lexical-extractor.md` | `candidates/lexical.md` |
| 3 | 叙事分析师 | `extractors/narrative-extractor.md` | `candidates/narrative.md` |
| 4 | 对话分析师 | `extractors/dialogic-extractor.md` | `candidates/dialogic.md` |
| 5 | 母题分析师 | `extractors/thematic-extractor.md` | `candidates/thematic.md` |
| 6 | 签名分析师 | `extractors/signature-extractor.md` | `candidates/signature.md` |

## 产出格式

每个 `candidates/<dimension>.md` 必须包含:

```markdown
# <维度名> 候选特征

## 特征列表

### 特征 1: <特征名>
- **描述**: 一句话描述
- **证据**: 从样本中引用的具体段落（≤150 字/段，至少 2 段）
- **强度**: 强 / 中 / 弱
- **可操作性**: 能转化为仿写指令吗？如何转化？
- **初步判断**: 可能是稳定特征 / 可能是偶发 / 可能是题材约束

### 特征 2: ...
...

## 跨维度观察

（该分析师注意到的、可能属于其他维度的特征，供阶段 2 交叉参考）
```

## 质量要求

- 每个维度至少产出 **5 条候选特征**
- 每条特征必须有**至少 2 段样本证据**
- 不确定的特征也要记录（标注"待验证"），不要自行淘汰
