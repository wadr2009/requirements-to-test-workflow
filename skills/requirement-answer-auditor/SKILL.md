---
name: requirement-answer-auditor
description: 检查需求问答过程是否完整，验证原始需求是否得到有效解决，将问题分类为已明确回答、推断回答和未回答类别。计算完成度并判定门控是否通过。
license: MIT
compatibility: 需要需求文档和问答记录。
metadata:
  author: sangang
  version: "2.1"
---

# 需求问答审核器 —— 门控判定与完成度计算

## 触发条件

- 阶段 1 第三步由编排器调用
- 用户直接要求"审核这个需求问答过程"

## 输入

| 参数 | 来源 | 获取方式 |
|------|------|----------|
| 解答报告 | `{OUTPUT_DIR}/01-answerer-解答报告.md` | `read_file` |
| 问题清单 | `{OUTPUT_DIR}/01-clarifier-问题清单.md` | `read_file` |
| 迭代范围（参考） | `{OUTPUT_DIR}/00-迭代范围.md` | `read_file` |

## 执行步骤

### Step 1：读取输入

```python
from hermes_tools import read_file

answer_report = read_file("{OUTPUT_DIR}/01-answerer-解答报告.md")["content"]
question_list = read_file("{OUTPUT_DIR}/01-clarifier-问题清单.md")["content"]
```

### Step 2：解析解答报告并交叉验证分类

从解答报告中提取关键数据，**同时运行分类一致性交叉验证**：

```python
import re

# 统计各分类数量
explicit = len(re.findall(r"分类.*文档明确回答", answer_report))
inferred = len(re.findall(r"分类.*文档推断回答", answer_report))
need_user = len(re.findall(r"分类.*需用户确认", answer_report))
total = explicit + inferred + need_user

# === v2.1 强制交叉验证 A：确定性: 中 ≠ 文档明确回答 ===
# 这是最常见的作弊方式 —— answerer 把推断标记为明确回答
medium_certainty = re.findall(r"\*\*确定性\*\*:\s*中", answer_report)
explicit_category = re.findall(r"\*\*分类\*\*:\s*文档明确回答", answer_report)
classification_cheat_count = 0
if len(medium_certainty) != len(inferred):
    classification_cheat_count = abs(len(medium_certainty) - len(inferred))
    # 将这标记为"分类作弊"而非静默接收

# === v2.1 强制交叉验证 B：每条文档明确回答必须有非空引用 ===
explicit_without_ref = 0
for match in re.finditer(r"\*\*分类\*\*:\s*文档明确回答", answer_report):
    context_after = answer_report[match.start():match.start()+500]
    basis = re.search(r"\*\*依据\*\*:\s*([^\n]+)", context_after)
    if not basis or not basis.group(1).strip() or basis.group(1).strip() in ("无", "无依据", "暂无"):
        explicit_without_ref += 1

# 计算真实完成度（仅文档明确回答计入，同 v2.0）
# 但如果在交叉验证中发现分类作弊，必须将作弊条目降级
completion = explicit / total * 100 if total > 0 else 0

# 检查问题总数与清单是否一致
question_expected = len(re.findall(r"\*\*问题\*\*:", question_list))
```

### Step 2.5：完成度真实性判定

审计报告的完成度必须以 Step 2 的精确计算结果为准，不得使用 answerer 报告中的声明值：

```python
# 如果发现分类作弊，必须降级作弊条目并重新计算
if classification_cheat_count > 0:
    true_explicit = explicit - classification_cheat_count
    true_completion = true_explicit / total * 100 if total > 0 else 0
    audit_warnings.append({
        "type": "分类作弊",
        "detail": f"发现 {classification_cheat_count} 条确定性:中的条目被错误标记为文档明确回答",
        "impact": f"真实完成度应为 {true_completion:.1f}%（而非声明的 {completion:.1f}%）",
        "action": "驳回解答报告，要求 answerer 修正分类后重新提交"
    })
```

### Step 3：逐条审核质量

对每条回答检查：

1. **是否直接回应问题**：答案是否真正回答了问题，而非绕弯子
2. **依据是否真实**：检查引用的文档出处是否真实存在
   ```python
   from hermes_tools import read_file
   full_req = read_file("{OUTPUT_DIR}/00-原始需求.md")["content"]
   # 对每条约定的引用，在原文中搜索验证
   ```
3. **分类是否正确**：
   - "文档明确回答" → 必须有具体文档引用
   - "文档推断回答" → 必须标注推断依据和"确定性: 中"
   - "需用户确认" → 文档中确实无相关信息

4. **推断回答是否过度**：
   - 如果文档中有足够信息可以直接回答，但被标记为"推断"，则标记为分类错误

### Step 4：生成审核报告并写入门控令牌

```python
from hermes_tools import write_file, read_file
import json

# 门控判定（v2.1：如有分类作弊，使用真实完成度）
true_explicit = explicit - classification_cheat_count if classification_cheat_count > 0 else explicit
true_completion = true_explicit / total * 100 if total > 0 else 0
gate_passed = true_completion >= 90
rounds = 当前轮次  # 从上下文中获取

if rounds >= 3 and not gate_passed:
    action = "BLOCKED：已超过 3 轮，暂停并展示阻塞清单"
elif gate_passed:
    action = "PASS：进入阶段 2"
else:
    action = "RETRY：回到 clarifier 生成补充问题"

# === v2.1 写入门控令牌文件（协议 1） ===
gate_token = {
    "stage": 1,
    "passed": gate_passed,
    "metrics": {
        "completion_rate": round(true_completion / 100, 2),
        "total_questions": total,
        "explicit_answers": true_explicit,
        "inferred_answers": inferred,
        "need_user_confirm": need_user,
        "classification_cheat_detected": classification_cheat_count > 0,
        "classification_cheat_count": classification_cheat_count,
        "explicit_without_ref": explicit_without_ref
    },
    "timestamp": datetime.now().isoformat()
}
write_file("{OUTPUT_DIR}/.gate-1.json", json.dumps(gate_token, ensure_ascii=False, indent=2))

# 强制回读验证
token_readback = read_file("{OUTPUT_DIR}/.gate-1.json")["content"]
assert len(token_readback) > 0, "FATAL: 门控令牌写入失败"
assert '"passed"' in token_readback, "FATAL: 门控令牌缺少 passed 字段"
```

### Step 5：写入审核报告

```python
write_file("{OUTPUT_DIR}/01-auditor-审核报告.md", report_content)
```

## 输出格式

输出文件 `{OUTPUT_DIR}/01-auditor-审核报告.md`：

```markdown
# 需求问答审核报告

**审核日期**: [日期]
**当前轮次**: 第 N 轮
**问题总数**: X个

---

## 完成度统计

| 分类 | 数量 | 占比 | 计入完成度 |
|------|------|------|-----------|
| 文档明确回答（确定性高） | {explicit} | {explicit/total*100}% | ✅ |
| 文档推断回答（确定性中） | {inferred} | {inferred/total*100}% | ❌ |
| 需用户确认（确定性低） | {need_user} | {need_user/total*100}% | ❌ |

**完成度**: {explicit}/{total} = **{completion}%**

---

## 门控判定

| 条件 | 当前值 | 阈值 | 结果 |
|------|--------|------|------|
| 完成度 | {completion}% | ≥ 90% | {'✅' if completion >= 90 else '❌'} |

**判定结果**: {PASS / RETRY / BLOCKED}

---

## 逐条审核

### 文档明确回答（{explicit}条）

| # | 问题 | 答案摘要 | 文档引用 | 审核 |
|---|------|----------|----------|------|
| 1 | {问题} | {摘要} | {引用} | ✅ 准确 |
| 2 | {问题} | {摘要} | {引用} | ⚠️ 引用不够具体 |

### 文档推断回答（{inferred}条）

| # | 问题 | 推断答案 | 推断依据 | 审核 |
|---|------|----------|----------|------|
| 1 | {问题} | {摘要} | {依据} | ⚠️ 推断合理但需确认 |
| 2 | {问题} | {摘要} | {依据} | ❌ 原文有直接答案，不应标记为推断 |

### 需用户确认（{need_user}条）

| # | 问题 | 优先级 | 建议 |
|---|------|--------|------|
| 1 | {问题} | 关键 | 建议用户确认后补充 |
| 2 | {问题} | 一般 | 可暂缓 |

---

## 分类错误（如有）

| # | 问题 | 当前分类 | 应为分类 | 原因 |
|---|------|----------|----------|------|
| 1 | {问题} | 文档推断回答 | 文档明确回答 | 需求文档第X节已明确说明 |

---

## 下一步行动

- {'✅ 门控通过 → 进入阶段 2（模块拆分）' if gate_passed else ''}
- {'❌ 门控未通过 → 回到 clarifier，聚焦以下未解决问题生成补充问题清单：' if not gate_passed else ''}
  - 推断回答项（{inferred}条）：需要更明确的文档证据
  - 需用户确认项（{need_user}条）：等待用户回答
- {'🚫 已阻塞：超过 3 轮仍未达标。请用户决策：' if rounds >= 3 and not gate_passed else ''}
  1. 人工补充未回答的问题
  2. 降低门控阈值（当前{completion}% → 目标阈值）
  3. 跳过未解决问题，继续后续阶段（风险自负）

---

## 阻塞清单（仅当 blocked 时）

以下问题无法通过文档回答，必须人工决策：

1. **{问题1}**（优先级：关键）
   阻塞原因：{原因}
   决策影响：{不决定会怎样}
   建议方案：{方案A / 方案B}

2. **{问题2}**（优先级：重要）
   ...
```

## 验证步骤

```python
from hermes_tools import read_file

audit = read_file("{OUTPUT_DIR}/01-auditor-审核报告.md")["content"]

# 1. 完成度计算正确
# 手动验证：explicit/total 是否匹配报告中的百分比

# 2. 门控判定与完成度一致
if "完成度: 100" in audit or "完成度: 9" in audit:  # 90%+
    assert "PASS" in audit or "✅" in audit, "高完成度但未通过门控"

# 3. 分类总数与问题总数一致
# 4. 如果有分类错误，必须列出
```

## 回环触发

如果门控判定为 RETRY，编排器将回到 clarifier。审核报告中的以下内容作为补充问题的输入：
- 推断回答项 → 作为"需要更明确证据"的方向
- 需用户确认项 → 如果用户已回答，更新为"文档明确回答"；如果未回答，继续保留

## 注意事项

- 计算完成度时**严格仅计入文档明确回答**（v2.0）
- 检查分类是否正确：推断回答不能混入明确回答
- 审核要客观：不要为了让门控通过而放宽标准
- 超过 3 轮必须阻塞，不要无限循环
- "需用户确认"项不阻塞流程，但会降低完成度
