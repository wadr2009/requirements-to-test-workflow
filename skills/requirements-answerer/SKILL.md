---
name: requirements-answerer
description: 基于提供的文档回答产品需求问题，严格遵循现有材料，为测试设计提供清晰简洁的答案。按三类处理问题：文档明确回答（计入完成度）、文档推断回答（不计入完成度，标注推断依据）、需用户确认（暂停等待）。
license: MIT
compatibility: 需要需求文档、设计文档和澄清问题清单。
metadata:
  author: sangang
  version: "2.0"
---

# 需求问题解答器 —— 基于文档回答澄清问题

## 触发条件

- 阶段 1 第二步由编排器调用
- 用户直接要求"根据需求文档回答这些问题"

## 输入

| 参数 | 来源 | 获取方式 |
|------|------|----------|
| 问题清单 | `{OUTPUT_DIR}/01-clarifier-问题清单.md` | `read_file` |
| 需求文档 | `{OUTPUT_DIR}/00-原始需求.md` | `read_file` |
| 迭代范围 | `{OUTPUT_DIR}/00-迭代范围.md` | `read_file` |
| 设计文档（可选） | 用户指定路径 | `read_file` |

## 执行步骤

### Step 1：读取所有输入

```python
from hermes_tools import read_file

questions = read_file("{OUTPUT_DIR}/01-clarifier-问题清单.md")["content"]
full_req = read_file("{OUTPUT_DIR}/00-原始需求.md")["content"]
scope = read_file("{OUTPUT_DIR}/00-迭代范围.md")["content"]
# design = read_file("docs/设计文档.md")["content"]  # 如有
```

### Step 2：解析问题清单

从问题清单中提取每个问题及其优先级。遍历所有问题，按以下流程逐条处理。

### Step 3：逐条回答（三类处理）

对每个问题，执行以下判定流程：

```python
# 对每个问题的处理逻辑（伪代码）
for question in questions:
    # 1. 在需求文档中搜索关键词
    #    用 search_files(pattern=关键词, path="{OUTPUT_DIR}", file_glob="00-*.md")
    # 2. 在设计文档中搜索（如有）
    #    用 search_files(pattern=关键词, path="docs/", file_glob="*.md")
    
    if 文档中有直接答案:
        # → 文档明确回答
        answer = 直接引用原文
        certainty = "高"
        category = "文档明确回答"
       计入完成度 = True
       
    elif 文档中有相关线索:
        # → 文档推断回答（不计入完成度！）
        answer = 基于线索推断 + 推断逻辑说明
        certainty = "中"
        category = "文档推断回答"
       计入完成度 = False  # v2.0 变更
       备注 = "需以实际实现为准"
       
    else:
        # → 需用户确认
        answer = "需要用户确认"
        certainty = "低"
        category = "需用户确认"
       计入完成度 = False
        # 暂停等待：使用 clarify 工具向用户提问
```

**关键原则（v2.0）**：

- 文档有线索时**可以**推断，但必须标注推断依据，且**不计入完成度**
- 仅当文档完全无信息时才标记"需用户确认"
- 推断回答将流转到下一轮澄清作为补充问题素材
- 不要为了凑完成度而把推断回答标记为"文档明确回答"

### Step 4：按模块组织答案

将答案按原问题的模块和优先级组织输出。

### Step 5：计算统计并写入

```python
from hermes_tools import write_file

# 统计
total = len(所有问题)
explicit = len(文档明确回答)
inferred = len(文档推断回答)
need_user = len(需用户确认)
completion = explicit / total * 100  # v2.0：推断不计入

write_file("{OUTPUT_DIR}/01-answerer-解答报告.md", report_content)
```

## 输出格式

输出文件 `{OUTPUT_DIR}/01-answerer-解答报告.md`：

```markdown
# 需求问题解答报告

**文档**: {ITERATION}迭代需求
**解答日期**: [日期]
**问题总数**: X个

---

## 模块一：{模块名称}

### 问题1（优先级：关键）

**问题**: {问题内容}
**答案**: {基于文档的准确答案}
**依据**: {文档引用，如"需求文档【270】第3.2节"或"设计文档接口定义第5行"}
**确定性**: 高
**分类**: 文档明确回答
**备注**: 无

### 问题2（优先级：重要）

**问题**: {问题内容}
**答案**: {基于线索的推断答案}
**依据**: {文档线索说明，如"需求文档提到'支持多银行'但未列具体银行列表，根据设计文档接口参数中的 bankCode 枚举推断"}
**确定性**: 中
**分类**: 文档推断回答
**备注**: 需以实际实现为准，建议用户确认

### 问题3（优先级：关键）

**问题**: {问题内容}
**答案**: 需要用户确认
**确定性**: 低
**分类**: 需用户确认
**备注**: 文档无相关信息，涉及业务规则决策

---

## 总结

| 指标 | 数量 | 占比 |
|------|------|------|
| 文档明确回答（确定性高） | {explicit} | {explicit/total*100}% |
| 文档推断回答（确定性中） | {inferred} | {inferred/total*100}% |
| 需用户确认（确定性低） | {need_user} | {need_user/total*100}% |
| **完成度**（仅明确回答） | **{explicit}** | **{completion}%** |

> 注：v2.0 规则下，推断回答不计入完成度。如完成度 < 90%，将触发补充澄清循环。
```

## 处理"需用户确认"项

对于分类为"需用户确认"的问题，回答完所有文档可回答的问题后，使用 `clarify` 工具逐批向用户提问：

```
clarify(
  question="以下问题需要您确认才能继续：\n1. {问题1}\n2. {问题2}\n请逐一回答或选择'稍后处理'",
  choices=["逐一回答", "稍后处理，先继续后续阶段", "跳过这些问题"]
)
```

用户回答后，用 `patch` 工具更新 `{OUTPUT_DIR}/01-answerer-解答报告.md`，将对应问题的分类改为"文档明确回答"（或"用户确认回答"），确定性改为"高"，并更新统计。

## 验证步骤

写入后验证：

```python
from hermes_tools import read_file
import re

report = read_file("{OUTPUT_DIR}/01-answerer-解答报告.md")["content"]

# 1. 每个问题都有答案、依据、确定性、分类
answer_count = len(re.findall(r"\*\*答案\*\*:", report))
basis_count = len(re.findall(r"\*\*依据\*\*:", report))
certainty_count = len(re.findall(r"\*\*确定性\*\*:", report))
category_count = len(re.findall(r"\*\*分类\*\*:", report))
assert answer_count == basis_count == certainty_count == category_count, \
    f"字段不完整: 答案{answer_count} 依据{basis_count} 确定性{certainty_count} 分类{category_count}"

# 2. 完成度计算正确
# 3. 没有"确定性: 中"被错误标记为"文档明确回答"
```

## 注意事项

- **严格基于文档**：不添加外部知识或主观猜测
- **推断必须标注**：推断回答必须注明推断逻辑和"确定性: 中"
- **推断不计入完成度**（v2.0）：这是防止"伪通过"门控的关键变更
- **文档引用要具体**：不能只写"需求文档"，要写"需求文档第X节"或引用原文
- **确定性低的暂停等待**：不要替用户做决策
- **推断回答将回环**：如果完成度不足，推断回答会成为下一轮澄清的重点
