---
name: requirements-to-test-workflow
description: >-
  顶层编排技能：将 8 个子技能串联为「需求分析与测试交付」完整流水线。
  从原始需求文档出发，经过澄清、模块拆分、测试点分析、测试用例生成四个阶段，
  最终产出 XMind 可导入的标准化测试用例。每个阶段内嵌质量门，不合格自动回环。
  支持全流程执行和单阶段断点续跑。大文档/复杂项目自动建议子代理并行。
  所有依赖子技能位于同目录下的 skills/ 子目录中。
license: MIT
metadata:
  author: sangang
  version: "4.1"
  pipeline_skills:
    - skills/requirements-clarifier
    - skills/requirements-answerer
    - skills/requirement-answer-auditor
    - skills/requirement-module-splitter
    - skills/requirements-test-point-analyzer
    - skills/test-feature-reviewer
    - skills/test-case-generator
    - skills/test-case-reviewer
  internal_skills:
    - skills/test-case-format
  sub_skills_dir: skills
---

# 需求分析与测试交付 —— 完整工作流

## 总览

将 8 个子技能编排为 4 个阶段，每阶段内部闭环（除非人工介入选择跳过）。
最终交付：**XMind 可导入的标准化测试用例** + 全链路审查报告。

```
┌──────────────────────────────────────────────────────────────────┐
│                   需求分析与测试交付流水线                          │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 阶段 1 — 需求澄清循环                                       │ │
│  │  ① clarifier ──▶ ② answerer ──▶ ③ auditor                  │ │
│  │  门控: 审核完成度 ≥ 90%                                       │ │
│  │  未通过: 回到 ① 出补充问题清单                                 │ │
│  └────────────────────────────────────────────────────────────┘ │
│           │                                                      │
│           ▼                                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 阶段 2 — 模块拆分                                           │ │
│  │  ④ module-splitter                                         │ │
│  │  产出: 按角色 + 目录树拆分的模块报告 + 依赖图                 │ │
│  └────────────────────────────────────────────────────────────┘ │
│           │                                                      │
│           ▼                                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 阶段 3 — 测试点分析循环                                      │ │
│  │  ⑤ test-point-analyzer ──▶ ⑥ test-feature-reviewer          │ │
│  │  门控: 覆盖度 ≥ 95% + 不一致项 0                             │ │
│  │  未通过: 回到 ⑤ 补充测试点                                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│           │                                                      │
│           ▼                                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 阶段 4 — 测试用例生成循环                                    │ │
│  │  ⑦ test-case-generator ──▶ ⑧ test-case-reviewer            │ │
│  │  门控: 覆盖度 100% + 格式合规 ≥ 95% + P0/P1 完整            │ │
│  │  未通过: 回到 ⑦ 修订用例                                    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  交付: XMind 用例文件 + 全阶段审查报告                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 强制验证协议（v4.1 硬约束）

> **以下规则不是建议，是不可绕过的硬约束。每条违反都必须立即停修，不得静默跳过。**

### 协议 1：门控令牌文件（Gate Token）

每个阶段完成后，必须将门控判定写入机器可解析的 JSON 文件。**下一阶段启动前，必须首先 `read_file` 读取上游令牌文件。** `passed=false` 时禁止进入下一阶段。

| 阶段 | 令牌文件 | `passed` 为 true 的条件 |
|------|---------|----------------------|
| 阶段 0 | `output/{ITERATION}/.gate-0.json` | 输出目录已创建 + `00-原始需求.md` 和 `00-迭代范围.md` 存在且非空 |
| 阶段 1 | `output/{ITERATION}/.gate-1.json` | `completion_rate >= 0.90`（仅 `分类:文档明确回答` 计入分子） |
| 阶段 3 | `output/{ITERATION}/.gate-3.json` | `coverage >= 0.95` 且 `inconsistencies == 0` |
| 阶段 4 | `output/{ITERATION}/.gate-4.json` | `coverage == 1.0` 且 `format_rate >= 0.95` 且 `p0p1_complete == true` 且 `traceability_ok == true` |

**令牌文件必须包含字段**: `{"stage": N, "passed": bool, "metrics": {...}, "timestamp": "..."}`

**门控判定必须用 `read_file` 从实际产出文件中提取数据计算，不得凭记忆。** 例如：

```python
from hermes_tools import read_file, write_file
import re, json

# 从文件读取数据 —— 不是从上下文中回忆
answerer = read_file("{OUTPUT_DIR}/01-answerer-解答报告.md")["content"]
explicit = len(re.findall(r"分类.*文档明确回答", answerer))
total_q = len(re.findall(r"\*\*问题\*\*:", answerer))
completion = explicit / total_q

gate = {
    "stage": 1,
    "passed": completion >= 0.90,
    "metrics": {"completion_rate": round(completion, 2), "total_questions": total_q, "explicit_answers": explicit},
    "timestamp": "{now}"
}
write_file("{OUTPUT_DIR}/.gate-1.json", json.dumps(gate, ensure_ascii=False, indent=2))
```

### 协议 2：写入后强制回读（Write-then-Read）

每次 `write_file` 产出文件后，**必须立即 `read_file`** 回读并验证：
1. 文件非空（`len(content) > 0`）
2. 结构完整（检查必要章节标题是否存在）
3. 数据自洽（统计数据与详表交叉验证）

**此步骤不是"建议"，是强制的。** 如果回读发现文件为空或结构不完整 → 立即重新生成，不得继续下一阶段。

### 协议 3：分类一致性强制检查

`确定性: 中` 的答案 **必须且只能** 对应 `分类: 文档推断回答`。
`确定性: 高` 的答案 **必须且只能** 对应 `分类: 文档明确回答`。

answerer 写入前、auditor 审核时均须用正则交叉验证此约束。**违反此约束的产出文件视为无效，必须修正后重新写入。**

### 协议 4：子代理的传递约束（并行模式）

向 `delegate_task` 传递 context 时，必须包含以下验证指令：
```
## 强制验证指令
1. 完成后写入 {OUTPUT_DIR}/module-{name}/.done.json，格式: {"status": "complete", "tc_count": N}
2. 在文件末尾必须包含"## 附录：全链路追溯矩阵（REQ → TP → TC）"
3. 追溯矩阵中每条 REQ 至少对应 1 行
4. 如果无法满足以上任何一条，在 .done.json 中写 "status": "blocked" 并说明原因
```

合并步骤在读取各模块产出前，先检查 `.done.json` 是否存在且 `status == "complete"`。

### 协议 5：检测到违规的响应

1. **立即停止**当前阶段，不继续下一步
2. 定位违规点并修复
3. 重新写入产出文件
4. 重新执行验证
5. 仅在验证全通过后进入下一阶段

**在任何情况下都不要将违规静默跳过。**

---


### 输出目录

所有阶段产出统一存放在项目根目录下的 `output/{迭代号}/` 目录中。

```
项目根目录/
└── output/
    └── {迭代号}/           ← 如 output/270/ 或 output/380/
        ├── README.md                    ← 全链路汇总（最后生成）
        ├── 00-原始需求.md               ← 从 Mockplus/文件提取的完整需求
        ├── 00-迭代范围.md               ← 经识别的当前迭代需求清单
        ├── 01-clarifier-问题清单.md      ← 阶段1-①：澄清问题
        ├── 01-answerer-解答报告.md       ← 阶段1-②：问题解答
        ├── 01-auditor-审核报告.md        ← 阶段1-③：审核 + 门控判定
        ├── 02-module-splitter-模块拆分.md ← 阶段2：模块拆分报告
        ├── 03-test-point-analyzer-测试点.md ← 阶段3-⑤：测试点分析
        ├── 03-test-feature-reviewer-评审.md ← 阶段3-⑥：评审 + 门控判定
        ├── 03-regression-scope.md          ← 阶段3.5：跨迭代回归范围
        ├── 04-test-case-generator-用例.md   ← 阶段4-⑦：XMind 用例
        ├── 04-test-case-reviewer-评审.md    ← 阶段4-⑧：评审 + 门控判定
        └── 99-可追溯性矩阵.md               ← 全链路追溯：REQ→TP→TC
```

### 阶段间数据传递（强制规则）

**文件是唯一可信数据源，禁止跳过文件在上下文间直接传递数据。**

- 每个阶段开始前，用 `read_file` 读取上游产出文件
- 每个阶段完成后，用 `write_file` 将结果写入对应文件
- 下游阶段通过文件路径引用上游产出，不依赖对话上下文中的临时数据
- 门控验证也必须从文件中读取数据计算，不得凭记忆

### 文件路径变量

在指令中使用以下变量，执行时替换为实际值：

| 变量 | 含义 | 示例 |
|------|------|------|
| `{OUTPUT_DIR}` | 输出目录 | `output/270/` |
| `{ITERATION}` | 迭代号 | `270` |
| `{PROJECT_ROOT}` | 项目根目录 | `.` (执行时的 cwd) |

### 断点续跑约定

如果某个阶段的产出文件已存在，则视为该阶段已完成，跳过执行。
如需重新执行某阶段，先删除对应产出文件或由用户明确指示。

---

## 阶段 0：准备工作

### 输入要求

- 需求文档（PRD / 迭代文档 / 产品规格说明）
- 后端设计文档 / 接口文档（**强烈推荐提供**，直接影响测试用例的精确性）

### 执行指令

#### Step 0-1：创建输出目录

使用 `terminal` 工具：

```bash
mkdir -p output/{ITERATION}/
```

#### Step 0-2：获取原始需求

**需求是本地文件**

使用 `read_file` 读取需求文档全文，将其复制到输出目录：

```python
from hermes_tools import read_file, write_file
content = read_file("docs/PRD.md")
write_file("{OUTPUT_DIR}/00-原始需求.md", content["content"])
```

#### Step 0-3：识别迭代范围（必做步骤）

需求文档可能包含多个迭代的完整需求，**必须先识别并界定当前迭代的测试范围**。

**执行方法**：

1. 用 `read_file` 读取 `{OUTPUT_DIR}/00-原始需求.md`
2. 识别迭代标注方式（如【270】、【220】、【380】等标注、章节标题中的迭代号）
3. 仅提取当前迭代标注的需求段落
4. 历史迭代内容仅作上下文参考，不纳入测试范围
5. 如果文档未明确标注迭代版本，使用 `clarify` 工具向用户确认

**产出文件**：使用 `write_file` 写入 `{OUTPUT_DIR}/00-迭代范围.md`：

```markdown
# {ITERATION}迭代需求范围

## 当前迭代范围（纳入测试）

### 模块一：XXX
- REQ-001: [功能点描述]
- REQ-002: [功能点描述]

### 模块二：XXX
- REQ-003: [功能点描述]
- ...

## 被排除的历史迭代内容（仅参考）

| 迭代号 | 内容 | 排除原因 |
|--------|------|----------|
| 【220】 | XXX功能 | 上一迭代已交付 |
```

> **REQ ID 编号规则**：REQ-001 起连续编号，贯穿全部模块。同一功能点在全流水线中保持唯一 ID。编号由阶段 0 首次分配后保持不变。

#### Step 0-5：分配 REQ ID（必做步骤）

阶段 0 结束时，必须为迭代范围内的**每个需求功能点**分配唯一 REQ ID：

```python
from hermes_tools import read_file, write_file
import re

scope = read_file("{OUTPUT_DIR}/00-迭代范围.md")["content"]

# 提取所有功能点，为其分配 REQ-001, REQ-002, ...
req_points = []
req_id = 1
for line in scope.split("\n"):
    if line.strip().startswith("- "):
        # 如果是功能点行，赋 REQ ID
        desc = line.strip()[2:]
        req_points.append(f"- REQ-{req_id:03d}: {desc}")
        req_id += 1

# 重新写入带 REQ ID 的文件
# ...（保持原有模块结构，为每个功能点添加 REQ 前缀）
```

**REQ ID 用途**：
- 阶段 2 模块拆分时，每个功能点带 REQ ID 引用
- 阶段 3 测试点分析时，每个 TP 标注来源 REQ
- 阶段 4 用例生成时，追溯矩阵记录 REQ→TP→TC 全链
- 阶段 4 评审时，验证每条 REQ 都有完整覆盖

**REQ ID 在后续阶段的引用方式**：
所有下游文件引用需求功能点时，必须同时使用 REQ ID 和功能描述。例如：
- `REQ-001: 工行审核通过后推送开立信息`
- 不可只用描述不用 ID，也不可只用 ID 不用描述

#### Step 0-4：阅读设计文档（强烈推荐）

如果提供了设计文档，**在阶段 1 之前完整阅读**：

```python
from hermes_tools import read_file
design_doc = read_file("docs/设计文档.md")  # 或用户指定的路径
# 或 read_file("docs/接口文档.md")
```

设计文档信息直接影响澄清质量和用例精确性。未读设计文档会导致大量本可自行回答的问题被错误标记为"需用户确认"。

### 门控验证

完成阶段 0 后，使用 `search_files` 验证，并写入门控令牌：

```python
from hermes_tools import search_files, write_file, read_file
import json

result = search_files(
    pattern="*.md",
    target="files",
    path="{OUTPUT_DIR}"
)
# 确认以下文件存在：
# - 00-原始需求.md
# - 00-迭代范围.md

# 强制回读验证文件非空
for f in ["00-原始需求.md", "00-迭代范围.md"]:
    content = read_file(f"{{OUTPUT_DIR}}/{f}")["content"]
    assert len(content) > 0, f"FATAL: {f} 为空"

# 写入门控令牌
gate_0 = {
    "stage": 0,
    "passed": True,
    "metrics": {"output_dir": "{OUTPUT_DIR}", "files_ready": ["00-原始需求.md", "00-迭代范围.md"]},
    "timestamp": datetime.now().isoformat()
}
write_file("{OUTPUT_DIR}/.gate-0.json", json.dumps(gate_0, ensure_ascii=False, indent=2))
```

### 触发方式

| 用户意图 | 执行策略 |
|----------|----------|
| "完整走一遍需求到测试" | 执行阶段 0 → 1 → 2 → 3 → 4 |
| "从模块拆分继续" | 检查 `01-auditor-审核报告.md` 门控通过 → 从阶段 2 开始 |
| "只看澄清结果" | 只执行阶段 0 + 阶段 1 |
| "已有澄清过了，直接拆分" | 跳过阶段 1，从阶段 2 开始 |

**大文档自动分流**：如果需求文档超 15 页或涉及 5+ 独立模块，阶段 2 完成后对每个模块使用 启动子代理 方式并行执行阶段 3+4。

---

## 阶段 1：需求澄清循环

### 目标

把模糊需求变成明确、可测试的答案，消除所有关键歧义。

### 执行指令

#### Step 1-1：生成澄清问题（子技能：requirements-clarifier）

**加载子技能**：读取 `skills/requirements-clarifier/SKILL.md` 并按其执行步骤操作。

**输入文件**：`{OUTPUT_DIR}/00-迭代范围.md`、`{OUTPUT_DIR}/00-原始需求.md`、设计文档（如有）

**关键步骤**：
1. `read_file("{OUTPUT_DIR}/00-迭代范围.md")` — 获取当前迭代范围
2. `read_file("{OUTPUT_DIR}/00-原始需求.md")` — 获取完整需求背景
3. 分析需求，按优先级（关键/重要/一般）分类生成问题清单
4. `write_file("{OUTPUT_DIR}/01-clarifier-问题清单.md")` — 写入问题清单

**产出**：`{OUTPUT_DIR}/01-clarifier-问题清单.md`

#### Step 1-2：回答问题（子技能：requirements-answerer）

**加载子技能**：读取 `skills/requirements-answerer/SKILL.md` 并按其执行步骤操作。

**输入文件**：`{OUTPUT_DIR}/01-clarifier-问题清单.md`、`{OUTPUT_DIR}/00-原始需求.md`、设计文档

**关键步骤**：
1. `read_file("{OUTPUT_DIR}/01-clarifier-问题清单.md")` — 读取问题
2. `read_file("{OUTPUT_DIR}/00-原始需求.md")` — 对照原文
3. 逐条回答，按三类处理（详见 answerer 子技能）
4. 推断回答必须标注"确定性: 中"和推断依据，**不计入已完成**
5. `write_file("{OUTPUT_DIR}/01-answerer-解答报告.md")` — 写入解答

**产出**：`{OUTPUT_DIR}/01-answerer-解答报告.md`

#### Step 1-3：审核解答（子技能：requirement-answer-auditor）

**加载子技能**：读取 `skills/requirement-answer-auditor/SKILL.md` 并按其执行步骤操作。

**输入文件**：`{OUTPUT_DIR}/01-answerer-解答报告.md`、`{OUTPUT_DIR}/01-clarifier-问题清单.md`

**关键步骤**：
1. `read_file("{OUTPUT_DIR}/01-answerer-解答报告.md")` — 读取解答
2. `read_file("{OUTPUT_DIR}/01-clarifier-问题清单.md")` — 对照原始问题
3. 计算完成度、分类统计、生成审核报告
4. `write_file("{OUTPUT_DIR}/01-auditor-审核报告.md")` — 写入审核报告

**产出**：`{OUTPUT_DIR}/01-auditor-审核报告.md`

### answerer 边界规则

answerer 回答问题时，**必须按以下三类处理每个问题**，不得混用：

| 分类 | 判定条件 | 处理方式 | 计入完成度 |
|------|---------|---------|-----------|
| **文档明确回答** | 需求文档或设计文档中有直接、明确的答案 | 直接回答，标注"确定性: 高"，引用文档出处 | ✅ 是 |
| **文档推断回答** | 文档有相关线索但未直接说明 | 回答并标注"确定性: 中"+"推断依据"+"需以实际实现为准" | ❌ **否**（v4.0 修改） |
| **需用户确认** | 文档完全无信息，或涉及产品决策 | 标注"确定性: 低"和"需要用户确认"，暂停等待 | ❌ 否 |

**v4.0 关键变更**：推断回答不再计入完成度分子。只有文档明确回答才算已完成。推断回答流转到下一轮澄清作为补充问题素材。

### 完成度计算公式（v4.0）

```
完成度 = 文档明确回答数 / 问题总数 × 100%
```

### 门控规则

```python
from hermes_tools import read_file
import json

# v2.1：进入阶段 2 前，必须读取阶段 1 的门控令牌
gate_1 = json.loads(read_file("{OUTPUT_DIR}/.gate-1.json")["content"])
if not gate_1["passed"]:
    # 门控未通过 —— 禁止进入阶段 2
    raise RuntimeError(
        f"阶段 1 门控未通过（完成度: {gate_1['metrics']['completion_rate']*100:.0f}%）。"
        f"必须回到 Step 1-1 修复后重试。"
    )
```

```
审核完成度 ≥ 90%  → ✅ 通过，进入阶段 2
审核完成度 < 90%  → ❌ 回到 Step 1-1，输入「未回答 + 推断回答 + 需用户确认」生成补充问题
超过 3 轮仍未 ≥ 90% → 暂停并展示阻塞清单，请用户决策（人工补充 / 降低阈值 / 跳过）
```

**回环执行**：如果门控未通过，再次执行 Step 1-1 时，问题清单应聚焦于上一轮未解决的问题（推断回答 + 需用户确认项），而非重新生成全套问题。

### 人工介入点

- "需用户确认"的问题必须使用 `clarify` 工具暂停等待用户回答
- auditor 报告中"需用户确认"的问题，不计入完成度，标记为"待用户确认"
- 用户回答后，将答案补充到 `01-answerer-解答报告.md`，重新运行 auditor

---

## 阶段 2：模块拆分

### 目标

将需求按用户角色 + 功能目录拆分为可独立测试的模块，理清依赖关系。

### 执行指令（子技能：requirement-module-splitter）

**加载子技能**：读取 `skills/requirement-module-splitter/SKILL.md` 并按其执行步骤操作。

**输入文件**：`{OUTPUT_DIR}/00-迭代范围.md`、`{OUTPUT_DIR}/01-auditor-审核报告.md`

**关键步骤**：
1. `read_file("{OUTPUT_DIR}/00-迭代范围.md")` — 获取需求范围
2. `read_file("{OUTPUT_DIR}/01-auditor-审核报告.md")` — 获取澄清后的需求理解
3. 按用户角色拆分功能
4. 生成目录树结构（深度 ≤ 4 层）
5. 分析模块依赖关系，标记循环依赖
6. 为每个模块标注风险等级（高/中/低）
7. `write_file("{OUTPUT_DIR}/02-module-splitter-模块拆分.md")` — 写入拆分报告

**产出**：`{OUTPUT_DIR}/02-module-splitter-模块拆分.md`

### 门控验证

在阶段 2 完成后，执行以下验证：

```python
from hermes_tools import read_file

report = read_file("{OUTPUT_DIR}/02-module-splitter-模块拆分.md")["content"]

# 检查项：
# 1. 所有需求功能点都归入至少一个模块 → 检查报告中功能点总数与需求一致
# 2. 依赖图无循环引用 → 检查报告中"循环依赖"标记为空
# 3. 风险等级标注完整 → 检查每个模块都有高/中/低标记
```

- 任一检查未通过 → 修正后重新写入
- 全部通过 → 进入阶段 3

---

## 阶段 3：测试点分析循环

### 目标

从需求中提取结构化测试点，分析依赖和影响范围，确保无遗漏。

### 前置检查（v2.1 强制）

进入阶段 3 前，**必须先读取阶段 1 和阶段 2 的门控令牌**：

```python
from hermes_tools import read_file
import json

# 验证阶段 1 门控已通过
gate_1 = json.loads(read_file("{OUTPUT_DIR}/.gate-1.json")["content"])
assert gate_1["passed"], f"FATAL: 阶段 1 门控未通过，禁止进入阶段 3"

# 验证阶段 2 产出文件存在
module_report = read_file("{OUTPUT_DIR}/02-module-splitter-模块拆分.md")["content"]
assert len(module_report) > 0, "FATAL: 模块拆分报告为空"
```

### 执行指令

#### Step 3-1：生成测试点（子技能：requirements-test-point-analyzer）

**加载子技能**：读取 `skills/requirements-test-point-analyzer/SKILL.md` 并按其执行步骤操作。

**输入文件**：`{OUTPUT_DIR}/00-迭代范围.md`、`{OUTPUT_DIR}/02-module-splitter-模块拆分.md`

**关键步骤**：
1. `read_file("{OUTPUT_DIR}/00-迭代范围.md")` — 获取需求
2. `read_file("{OUTPUT_DIR}/02-module-splitter-模块拆分.md")` — 获取模块结构
3. 对每个模块提取测试点（编号 TP001…TPnnn）
4. 分析功能依赖关系
5. 识别需求变更对现有功能的影响范围
6. `write_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md")` — 写入测试点

**产出**：`{OUTPUT_DIR}/03-test-point-analyzer-测试点.md`

#### Step 3-2：评审测试点（子技能：test-feature-reviewer）

**加载子技能**：读取 `skills/test-feature-reviewer/SKILL.md` 并按其执行步骤操作。

**输入文件**：`{OUTPUT_DIR}/03-test-point-analyzer-测试点.md`、`{OUTPUT_DIR}/00-迭代范围.md`

**关键步骤**：
1. `read_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md")` — 读取测试点
2. `read_file("{OUTPUT_DIR}/00-迭代范围.md")` — 对照需求
3. 计算覆盖度
4. 检查一致性和可测试性
5. `write_file("{OUTPUT_DIR}/03-test-feature-reviewer-评审.md")` — 写入评审报告

**产出**：`{OUTPUT_DIR}/03-test-feature-reviewer-评审.md`

### 门控规则

从 `03-test-feature-reviewer-评审.md` 中提取门控指标：

```
覆盖度 ≥ 95% 且 不一致项 = 0  → ✅ 通过，进入阶段 4
覆盖度 < 95% 或 不一致项 > 0  → ❌ 回到 Step 3-1，根据 reviewer 报告补充/修正
```

### 评审重点

- **覆盖完整性**：每个需求功能点都有对应测试点
- **一致性**：测试点方向与需求描述无矛盾
- **可测试性**：每个测试点有明确的前置条件 + 预期结果

---

## 阶段 3.5：跨迭代回归分析

### 目标

如果存在历史迭代的测试产出，分析当前迭代变更对历史功能的影响，识别需要回归验证的测试点和用例。

### 触发条件

阶段 3 门控通过后，自动检测是否存在历史迭代输出目录：

```python
from hermes_tools import search_files

# 检测 output/ 下是否存在其他迭代目录
all_dirs = search_files(
    pattern="*",
    target="files",
    path="output/"
)

# 筛选出非当前迭代的目录
historical_iterations = [d for d in all_dirs 
    if d.startswith("output/") and d != f"output/{ITERATION}/"]
```

- 如果存在历史迭代 → 执行阶段 3.5
- 如果不存在 → 跳过，直接进入阶段 4

### 执行指令

#### Step 3.5-1：加载历史迭代的关键产出

```python
from hermes_tools import read_file
import re

# 对每个历史迭代，读取其测试点和用例
regression_targets = []

for hist_iter in historical_iterations:
    # 读取历史测试点
    hist_tps = read_file(f"{hist_iter}/03-test-point-analyzer-测试点.md")
    # 读取历史用例
    hist_tcs = read_file(f"{hist_iter}/04-test-case-generator-用例.md")
    # 读取历史 REQ 清单
    hist_reqs = read_file(f"{hist_iter}/00-迭代范围.md")
    
    regression_targets.append({
        "iteration": hist_iter,
        "tps": hist_tps["content"],
        "tcs": hist_tcs["content"],
        "reqs": hist_reqs["content"]
    })
```

#### Step 3.5-2：提取当前迭代的变更点

从当前迭代需求和设计文档中提取变更类型：

```python
# 从 00-迭代范围.md 和设计文档中识别变更
current_changes = {
    "状态机变更": [],      # 状态流转逻辑的修改
    "接口变更": [],        # API 路径、入参、出参的修改
    "数据表变更": [],      # 表结构、字段、枚举值的修改
    "业务规则变更": [],    # 额度、阈值、校验规则的修改
    "新增功能": [],        # 全新功能（不触发回归，但可能影响旧功能的调用方）
}
```

**变更识别方法**：
- 对比 `00-原始需求.md` 中的迭代标注，识别标注为当前迭代的新增/修改描述
- 从设计文档中提取 "变更说明"、"修改记录" 等章节
- 如果无法自动识别，使用 `clarify` 工具向用户确认关键变更点

#### Step 3.5-3：计算影响范围

对每个历史迭代的功能点，评估是否受当前变更影响：

```python
affected_historical_items = []

for target in regression_targets:
    hist_tp_list = parse_test_points(target["tps"])
    
    for tp in hist_tp_list:
        affected = False
        reason = []
        
        # 检查 1：状态机变更影响
        if 历史TP的状态与当前变更的状态机重叠:
            affected = True
            reason.append("状态机变更")
        
        # 检查 2：接口变更影响
        if 历史TP调用的接口在当前变更中被修改:
            affected = True
            reason.append("接口变更")
        
        # 检查 3：数据表变更影响
        if 历史TP操作的数据表在当前变更中被修改:
            affected = True
            reason.append("数据表变更")
        
        # 检查 4：业务规则变更影响
        if 历史TP依赖的业务规则在当前变更中被修改:
            affected = True
            reason.append("业务规则变更")
        
        if affected:
            affected_historical_items.append({
                "iteration": target["iteration"],
                "tp_id": tp["id"],
                "tp_desc": tp["description"],
                "affected_reasons": reason,
                "impact_level": "高" if "状态机" in reason or "接口" in reason else "中",
                "suggested_action": "重新执行" if "状态机" in reason else "抽查验证"
            })
```

#### Step 3.5-4：生成回归范围报告

```python
from hermes_tools import write_file

write_file("{OUTPUT_DIR}/03-regression-scope.md", regression_report)
```

### 产出

`{OUTPUT_DIR}/03-regression-scope.md`

### 如何影响后续阶段

阶段 4 生成测试用例时：
1. 除了为当前迭代的测试点生成用例外，还要为 `03-regression-scope.md` 中的受影响历史测试点生成回归用例
2. 回归用例在 XMind 文件中单独分组：`## 回归测试`
3. 回归用例的优先级 = 原历史用例优先级（不降级）
4. 回归用例在追溯矩阵中标注来源迭代

### 输出格式

```markdown
# 跨迭代回归分析报告

**当前迭代**: {ITERATION}
**分析日期**: [日期]

---

## 一、当前迭代变更摘要

| 变更类型 | 变更内容 | 影响范围 |
|----------|----------|----------|
| 状态机变更 | 审核流程增加"预审"节点 | 所有依赖审核状态的模块 |
| 接口变更 | /api/audit/submit 新增 bankCode 必填字段 | 审核提交的所有调用方 |
| 数据表变更 | audit_record 表新增 pre_audit_status 字段 | 审核记录的读写操作 |

---

## 二、受影响的回归范围

### 历史迭代：output/270/

| 受影响 TP | 描述 | 影响原因 | 影响等级 | 建议操作 |
|-----------|------|----------|----------|----------|
| TP015 | 审核提交-字段校验 | 接口变更（新增必填字段） | 高 | 重新执行 |
| TP016 | 审核通过-状态通知 | 状态机变更（新增预审节点） | 高 | 重新执行 |
| TP020 | 审核列表查询 | 数据表变更（新增字段） | 中 | 抽查验证 |
| TP025 | 签约管理-额度检查 | 业务规则变更（额度计算逻辑） | 中 | 抽查验证 |

### 历史迭代：output/220/

| 受影响 TP | 描述 | 影响原因 | 影响等级 | 建议操作 |
|-----------|------|----------|----------|----------|
| TP008 | 授信额度查询 | 接口变更 | 中 | 抽查验证 |

---

## 三、回归测试统计

| 历史迭代 | 受影响 TP 数 | 需重新执行 | 需抽查验证 |
|----------|-------------|-----------|-----------|
| output/270/ | 4 | 2 | 2 |
| output/220/ | 1 | 0 | 1 |
| **合计** | **5** | **2** | **3** |

---

## 四、回归测试建议

1. **优先执行高影响项**：TP015（接口变更）、TP016（状态机变更）
2. **关注数据兼容性**：验证新字段对历史数据的影响
3. **接口向后兼容检查**：确认新增必填字段不影响已有调用方
4. **建议在阶段 4 中为这些回归项生成独立的回归测试用例**
```

---

## 阶段 4：测试用例生成循环

### 目标

将测试点转化为可直接执行的标准化测试用例，输出 XMind 可导入文件。

### 前置检查（v2.1 强制）

进入阶段 4 前，**必须先读取阶段 3 的门控令牌**：

```python
from hermes_tools import read_file
import json

# 验证阶段 3 门控已通过
gate_3 = json.loads(read_file("{OUTPUT_DIR}/.gate-3.json")["content"])
assert gate_3["passed"], (
    f"FATAL: 阶段 3 门控未通过"
    f"（覆盖度: {gate_3['metrics']['coverage']*100:.0f}%, "
    f"不一致项: {gate_3['metrics']['inconsistencies']}）。"
    f"禁止进入阶段 4，必须回到阶段 3 修复。"
)
```

### 执行指令

#### Step 4-1：生成测试用例（子技能：test-case-generator）

**加载子技能**：读取 `skills/test-case-generator/SKILL.md` 并按其执行步骤操作。

**前置要求**：生成用例前，**必须先加载 `skills/test-case-format/SKILL.md`** 获取 XMind 格式规范。格式规范是格式唯一权威来源。

**输入文件**：`{OUTPUT_DIR}/00-迭代范围.md`、`{OUTPUT_DIR}/03-test-point-analyzer-测试点.md`、`{OUTPUT_DIR}/03-test-feature-reviewer-评审.md`、`{OUTPUT_DIR}/03-regression-scope.md`（如存在）、设计文档

**关键步骤**：
1. 使用 `skill_view` 或 `read_file` 加载 `skills/test-case-format/SKILL.md`（**必须先做**）
2. `read_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md")` — 获取测试点
3. `read_file("{OUTPUT_DIR}/00-迭代范围.md")` — 获取需求原文
4. `read_file("{OUTPUT_DIR}/03-regression-scope.md")` — 获取回归范围（如文件存在）
5. 对每个测试点生成标准化用例
6. 如存在回归范围，在文件末尾增加 `## 回归测试` 分组，为受影响的回归测试点生成回归用例
7. 严格遵循 test-case-format 的层级规范
8. **默认输出为一个合并文件**，不按模块拆分
9. `write_file("{OUTPUT_DIR}/04-test-case-generator-用例.md")` — 写入用例

**产出**：`{OUTPUT_DIR}/04-test-case-generator-用例.md`

#### Step 4-2：评审测试用例（子技能：test-case-reviewer）

**加载子技能**：读取 `skills/test-case-reviewer/SKILL.md` 并按其执行步骤操作。

**输入文件**：`{OUTPUT_DIR}/04-test-case-generator-用例.md`、`{OUTPUT_DIR}/00-迭代范围.md`、`{OUTPUT_DIR}/03-test-point-analyzer-测试点.md`

**关键步骤**：
1. `read_file("{OUTPUT_DIR}/04-test-case-generator-用例.md")` — 读取用例
2. `read_file("{OUTPUT_DIR}/00-迭代范围.md")` — 对照需求
3. `read_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md")` — 对照测试点
4. 计算覆盖度、格式合规率
5. 检查 P0/P1 完整性
6. `write_file("{OUTPUT_DIR}/04-test-case-reviewer-评审.md")` — 写入评审报告

**产出**：`{OUTPUT_DIR}/04-test-case-reviewer-评审.md`

### 门控规则（最终质量门）

从 `04-test-case-reviewer-评审.md` 中提取门控指标：

```
覆盖度 = 100% 且 格式合规 ≥ 95% 且 P0/P1 完整 且 追溯链完整  → ✅ 交付
任一条件不满足                                          → ❌ 回到 Step 4-1 修订用例
```

### 输出文件组织方式

**默认输出为一个合并文件**，不按模块拆分。

**标题层级规范**（保证只有一个一级标题）：

| 层级 | Markdown | 内容 | 示例 |
|------|----------|------|------|
| 一级 | `#` | 迭代名称（仅一个） | `# 270迭代测试用例` |
| 二级 | `##` | 功能模块 | `## 一、保理融资审核-单笔审核` |
| 三级 | `###` | 功能分组 | `### 1.工行开立信息上送` |
| 四级 | `####` | 测试用例声明 | `#### tc-P0：工行审核通过-开立信息首次推送成功` |
| 五级 | `#####` | 前置条件 / 测试步骤 | `##### pc：...` |
| 六级 | `######` | 预期结果 | `###### 结果1;结果2` |

> 当功能分组层级较深时，用例声明的 `###` 可向后延伸至 `####` 或更深层级。

### 交付物清单

阶段 4 通过后，生成 `{OUTPUT_DIR}/README.md` 作为全链路汇总：

```python
from hermes_tools import read_file, write_file

# 汇总各阶段关键信息
summary = f"""# {ITERATION}迭代测试交付汇总

## 阶段 1 澄清结果
{read_file("{OUTPUT_DIR}/01-auditor-审核报告.md")["content"]}

## 阶段 2 模块拆分
{read_file("{OUTPUT_DIR}/02-module-splitter-模块拆分.md")["content"]}

## 阶段 3 测试点
- 测试点分析: {OUTPUT_DIR}/03-test-point-analyzer-测试点.md
- 评审结果: {OUTPUT_DIR}/03-test-feature-reviewer-评审.md

## 阶段 4 测试用例
- 用例文件: {OUTPUT_DIR}/04-test-case-generator-用例.md ← XMind 导入用此文件
- 评审结果: {OUTPUT_DIR}/04-test-case-reviewer-评审.md

## 可追溯性矩阵
- 全链路追溯: {OUTPUT_DIR}/99-可追溯性矩阵.md ← REQ→TP→TC 完整映射

## 门控结果
- 阶段 1: {'✅ 通过' if stage1_pass else '❌ 未通过'}
- 阶段 2: {'✅ 通过' if stage2_pass else '❌ 未通过'}
- 阶段 3: {'✅ 通过' if stage3_pass else '❌ 未通过'}
- 阶段 4: {'✅ 通过' if stage4_pass else '❌ 未通过'}
"""
write_file("{OUTPUT_DIR}/README.md", summary)
```

最终交付物清单：
1. **XMind 可导入文件**：`{OUTPUT_DIR}/04-test-case-generator-用例.md`
2. **全链路汇总**：`{OUTPUT_DIR}/README.md`
3. **可追溯性矩阵**：`{OUTPUT_DIR}/99-可追溯性矩阵.md`（REQ→TP→TC 全链路映射）
4. **全部中间产出**：阶段 1→4 的所有文件

### 可追溯性矩阵生成

阶段 4 通过后，从各阶段产出中提取数据生成 `{OUTPUT_DIR}/99-可追溯性矩阵.md`：

```python
from hermes_tools import read_file, write_file
import re

# 1. 从 00-迭代范围.md 提取所有 REQ
scope = read_file("{OUTPUT_DIR}/00-迭代范围.md")["content"]
reqs = re.findall(r"REQ-(\d{3}):\s*(.+)", scope)
# → [(001, "工行审核通过后推送开立信息"), (002, "..."), ...]

# 2. 从 03-test-point-analyzer-测试点.md 提取 REQ→TP 映射
tp_report = read_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md")["content"]
# 解析每个测试点的"来源需求"字段建立 REQ→TP 映射

# 3. 从 04-test-case-generator-用例.md 提取 TP→TC 映射
tc_report = read_file("{OUTPUT_DIR}/04-test-case-generator-用例.md")["content"]
# 从附录的追溯矩阵中提取 TP→TC 映射

# 4. 组装三维矩阵
matrix = []
for req_id, req_desc in reqs:
    tps = find_tps_for_req(req_id)    # REQ→TP
    tcs = find_tcs_for_tps(tps)       # TP→TC
    p0_count = sum(1 for tc in tcs if "P0" in tc)
    p1_count = sum(1 for tc in tcs if "P1" in tc)
    matrix.append({
        "req_id": f"REQ-{req_id}",
        "req_desc": req_desc,
        "tps": tps,
        "tcs": tcs,
        "p0": p0_count,
        "p1": p1_count,
        "covered": len(tcs) > 0
    })

# 5. 写入可追溯性矩阵
write_file("{OUTPUT_DIR}/99-可追溯性矩阵.md", matrix_content)
```

矩阵文件格式见下方 `99-可追溯性矩阵.md` 输出模板。

---

## 并行策略

当阶段 2 产出 N 个独立模块时，可按模块并行执行阶段 3+4。**并行模式下 REQ ID 需要命名空间隔离**，避免不同子代理生成冲突的 ID。

### REQ ID 命名空间规则（并行模式）

在并行模式下，阶段 0 的全局 REQ ID 分配不再适用。改为以下三层命名空间：

| 命名空间 | 格式 | 用途 | 示例 |
|----------|------|------|------|
| **模块 REQ** | `REQ-{模块前缀}{序号}` | 模块内部的功能点 | `REQ-A01`, `REQ-A02`, `REQ-B01` |
| **集成 REQ** | `REQ-X{序号}` | 跨模块的集成功能点 | `REQ-X01`（模块A和B的交集） |
| **回归 REQ** | `REQ-R{序号}` | 回归测试项 | `REQ-R01` |

### 并行前准备：分配 REQ 命名空间

阶段 2 完成后，在启动子代理之前，编排器必须完成 REQ 命名空间分配：

```python
from hermes_tools import read_file, write_file

modules_report = read_file("{OUTPUT_DIR}/02-module-splitter-模块拆分.md")["content"]

# 1. 解析模块列表
modules = parse_modules(modules_report)
# → [{"name": "审核管理", "prefix": "A"}, {"name": "额度管理", "prefix": "B"}, ...]

# 2. 为每个模块分配功能点 REQ 前缀
module_req_map = {}
for i, mod in enumerate(modules):
    prefix = chr(65 + i)  # A, B, C, ...
    module_req_map[mod["name"]] = {
        "prefix": prefix,
        "req_start": 1,
        "req_count": len(mod["功能点"]),
        "req_ids": [f"REQ-{prefix}{j+1:02d}" for j in range(len(mod["功能点"]))]
    }

# 3. 识别跨模块集成点，分配 REQ-X ID
integration_points = identify_cross_module_points(modules)
x_req_ids = [f"REQ-X{j+1:02d}" for j in range(len(integration_points))]

# 4. 将命名空间映射写入文件，供子代理和合并步骤使用
write_file("{OUTPUT_DIR}/00-req-namespace.json", json.dumps({
    "modules": module_req_map,
    "integration": x_req_ids,
    "mode": "parallel"
}))
```

### 并行执行

```
阶段 2 产出 N 个模块
         │
         ├─ 模块 A → delegate_task goal="对模块A执行阶段3+4，REQ前缀=REQ-A，输出到 module-A/"
         ├─ 模块 B → delegate_task goal="对模块B执行阶段3+4，REQ前缀=REQ-B，输出到 module-B/"
         └─ 模块 C → delegate_task goal="对模块C执行阶段3+4，REQ前缀=REQ-C，输出到 module-C/"
         │
         └─ 集成代理 → delegate_task goal="对跨模块集成点执行阶段3+4，REQ前缀=REQ-X"
                              │
                              ▼
                    合并 + REQ ID 统一 + 去重 + 最终审查
```

### 子代理 context 模板（并行模式）

```python
context = f"""
## 任务
对模块「{module_name}」执行测试点分析（阶段3）+ 测试用例生成（阶段4），严格按 test-case-format 格式输出。

## REQ ID 命名空间（必须遵守！）
- 本模块功能点使用前缀：REQ-{prefix}
- 编号从 REQ-{prefix}01 开始连续编号
- 测试点 ID 使用模块前缀：TP-{prefix}001, TP-{prefix}002...
- 用例在追溯矩阵中必须使用带前缀的 REQ ID

## 模块信息
{module_description}

## 原始需求
{read_file("{OUTPUT_DIR}/00-迭代范围.md")["content"]}

## 设计要求
1. 先加载 skills/test-case-format/SKILL.md 获取格式规范
2. 输出一个合并的 XMind 格式 Markdown 文件
3. 严格按标题层级
4. 输出中文
5. 完成后用 write_file 写入 {OUTPUT_DIR}/module-{module_name}/04-test-case-generator-用例.md
6. 测试点分析写入 {OUTPUT_DIR}/module-{module_name}/03-test-point-analyzer-测试点.md
7. 追溯矩阵必须包含 REQ ID（带前缀）
"""
```

### 合并步骤（处理 REQ ID 统一）

```python
from hermes_tools import read_file, write_file, search_files
import re, json

# 1. 读取命名空间映射
namespace = json.loads(read_file("{OUTPUT_DIR}/00-req-namespace.json")["content"])

# 2. 找到所有模块的用例和测试点文件
module_tc_files = search_files(
    pattern="04-test-case-generator-用例.md",
    target="files",
    path="{OUTPUT_DIR}/module-"
)
module_tp_files = search_files(
    pattern="03-test-point-analyzer-测试点.md",
    target="files",
    path="{OUTPUT_DIR}/module-"
)

# 3. 建立模块前缀 → 全局 REQ ID 的映射（用于最终统一）
# 格式: {"REQ-A01": "REQ-001", "REQ-A02": "REQ-002", "REQ-B01": "REQ-003", ...}
global_req_counter = 1
prefix_to_global = {}

for mod_name, mod_info in namespace["modules"].items():
    for req_id in mod_info["req_ids"]:
        prefix_to_global[req_id] = f"REQ-{global_req_counter:03d}"
        global_req_counter += 1

# 集成 REQ 也纳入全局映射
for x_id in namespace["integration"]:
    prefix_to_global[x_id] = f"REQ-{global_req_counter:03d}"
    global_req_counter += 1

# 4. 合并用例文件
merged = f"# {ITERATION}迭代测试用例\n\n"

for mod_file in module_tc_files:
    content = read_file(mod_file["path"])["content"]
    
    # 替换模块前缀 REQ ID 为全局 REQ ID
    for prefix_id, global_id in prefix_to_global.items():
        content = content.replace(prefix_id, global_id)
    
    merged += content + "\n\n"

# 5. 合并测试点文件（同样替换 REQ ID）
merged_tps = ""
for tp_file in module_tp_files:
    content = read_file(tp_file["path"])["content"]
    for prefix_id, global_id in prefix_to_global.items():
        content = content.replace(prefix_id, global_id)
    merged_tps += content + "\n\n"

# 6. 合并追溯矩阵
# 从各模块的附录中提取 REQ→TP→TC 行，统一写入合并文件的附录

# 7. 处理跨模块集成测试点（不丢失）
# 集成代理产出的测试点和用例单独处理

# 8. 写入合并文件
write_file("{OUTPUT_DIR}/04-test-case-generator-用例.md", merged)
write_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md", merged_tps)

# 9. 重新运行阶段 4 评审
```

### REQ 命名空间映射文件格式

`{OUTPUT_DIR}/00-req-namespace.json`：

```json
{
  "mode": "parallel",
  "modules": {
    "审核管理": {
      "prefix": "A",
      "req_ids": ["REQ-A01", "REQ-A02", "REQ-A03"],
      "req_descriptions": {
        "REQ-A01": "工行审核通过后推送开立信息",
        "REQ-A02": "审核驳回后通知申请方",
        "REQ-A03": "额度不足时阻断申请提交"
      },
      "output_dir": "module-A/"
    },
    "额度管理": {
      "prefix": "B",
      "req_ids": ["REQ-B01", "REQ-B02"],
      "req_descriptions": {
        "REQ-B01": "额度查询接口",
        "REQ-B02": "额度变更记录"
      },
      "output_dir": "module-B/"
    }
  },
  "integration": ["REQ-X01"],
  "integration_descriptions": {
    "REQ-X01": "审核通过后额度扣减（跨审核+额度模块）"
  },
  "prefix_to_global_mapping": {
    "REQ-A01": "REQ-001",
    "REQ-A02": "REQ-002",
    "REQ-A03": "REQ-003",
    "REQ-B01": "REQ-004",
    "REQ-B02": "REQ-005",
    "REQ-X01": "REQ-006"
  }
}
```

### 子代理注意事项

- **REQ ID 命名空间隔离**：每个子代理使用独有前缀（REQ-A/B/C...），杜绝 ID 冲突
- **TP ID 同样带前缀**：`TP-A001`, `TP-A002`，避免合并时测试点 ID 冲突
- **跨模块集成测试点不丢失**：由专门的集成代理处理，使用 `REQ-X` 前缀
- **合并时统一 REQ ID**：按 `prefix_to_global_mapping` 将所有模块前缀 REQ 替换为全局连续 REQ
- **追溯矩阵必须包含 REQ 前缀**：方便合并时做前缀→全局映射
- **最终合并后仍需通过阶段 4 门控**：包括追溯链完整性检查

---

## 错误处理

### 文件缺失

如果某阶段输入文件不存在：

```python
from hermes_tools import search_files
result = search_files(pattern="*.md", target="files", path="{OUTPUT_DIR}")
# 如果缺少前置文件，报告用户并说明需要从哪个阶段开始
```

### 格式不合规

如果子技能产出不符合模板：
1. 用 `read_file` 读取产出文件
2. 对照子技能的"输出格式"模板逐项检查
3. 标记缺失字段，回到该子技能补充

### 回环死锁检测

如果同一阶段回环超过 3 次门控仍未通过：
1. 用 `write_file` 将阻塞清单写入 `{OUTPUT_DIR}/BLOCKED.md`
2. 使用 `clarify` 工具向用户展示阻塞项并请求决策

---

## 交付物模板

### 99-可追溯性矩阵.md 输出格式

```markdown
# 可追溯性矩阵 —— REQ → TP → TC 全链路

**迭代号**: {ITERATION}
**生成日期**: [日期]

---

## 概览

| 指标 | 数值 |
|------|------|
| 需求功能点总数（REQ） | {N} |
| 测试点总数（TP） | {M} |
| 测试用例总数（TC） | {K} |
| 已覆盖 REQ | {covered} |
| 未覆盖 REQ | {uncovered} |
| **需求覆盖率** | **{covered/N*100}%** |

---

## REQ → TP → TC 完整映射

| REQ ID | 需求功能点 | 测试点（TP） | P0 用例 | P1 用例 | P2 用例 | 覆盖状态 |
|--------|-----------|-------------|---------|---------|---------|----------|
| REQ-001 | 工行审核通过后推送开立信息 | TP001, TP002, TP003 | 2 | 1 | 0 | ✅ 完整 |
| REQ-002 | 审核驳回后通知申请方 | TP004, TP005 | 1 | 0 | 1 | ⚠️ 缺 P1 |
| REQ-003 | 额度不足时阻断申请提交 | TP006 | 1 | 1 | 0 | ✅ 完整 |
| ... | ... | ... | ... | ... | ... | ... |

---

## 未覆盖需求（如有）

| REQ ID | 需求功能点 | 缺失原因 | 建议 |
|--------|-----------|----------|------|
| REQ-010 | XXX功能 | 测试点分析遗漏 | 补充 TP 和 TC |

---

## 弱覆盖需求（仅 P2，无 P0/P1）

| REQ ID | 需求功能点 | P2 用例 | 风险 | 建议 |
|--------|-----------|---------|------|------|
| REQ-015 | XXX展示 | tc-P2-030 | 核心流程缺少冒烟测试 | 升级为 P0 |

---

## 覆盖统计

| 覆盖等级 | 数量 | REQ ID |
|----------|------|--------|
| ✅ 完整覆盖（P0+P1） | {a} | REQ-001, REQ-002, ... |
| ⚠️ 部分覆盖（仅 P1 或仅 P2） | {b} | REQ-005, ... |
| ❌ 未覆盖 | {c} | REQ-010, ... |

---

## 反向追溯：用例 → 需求

| 用例 | 优先级 | 覆盖的 TP | 覆盖的 REQ |
|------|--------|----------|-----------|
| tc-P0：工行审核通过-开立信息首次推送成功 | P0 | TP001 | REQ-001 |
| tc-P1：工行审核通过-开立信息补推 | P1 | TP002 | REQ-001 |
| tc-P0：审核驳回-短信通知 | P0 | TP004 | REQ-002 |
```

---

## 使用示例

**全流程**：
> "从这份 PRD 出发，完整走一遍需求到测试用例的流程，迭代号 270"

**断点续跑**：
> "需求已经澄清过了，从模块拆分继续，输出到 output/270/"

**特定阶段**：
> "只做测试点分析，不要生成用例"

**大项目并行**：
> "这个项目有 6 个模块，并行生成测试用例，最后合并"

---

## 注意事项

1. **文件是唯一可信数据源**：所有阶段间数据传递必须通过文件系统，不得依赖对话上下文
2. **阶段顺序不可跳过**：阶段 0（迭代范围识别+设计文档阅读）→ 阶段 1（澄清）→ 阶段 2（拆分）→ 阶段 3（测试点）→ 阶段 4（用例）
3. **门控是硬约束**：不合格 → 先修复，不要静默跳过
4. **推断 ≠ 已完成**（v4.0）：文档推断回答不计入完成度，必须标注推断依据并流转到下一轮澄清
5. **人工无法替代的决策点**要主动使用 `clarify` 提问，不要替用户做决定
6. **`test-case-format` 是格式唯一权威来源**，test-case-generator 执行前必须先加载该技能
7. **迭代范围必须先界定**：阶段 0 必须识别当前迭代标注并仅提取对应需求
8. **子代理隔离**：并行模式下每个子代理独立运行，合并时注意用例 ID 去重和跨模块测试点
9. **输出目录随迭代号变化**：每次执行使用不同的 `{ITERATION}` 避免覆盖历史产出
10. **可追溯性是硬指标**（v4.0）：每条 REQ 必须有 REQ→TP→TC 的完整追溯链，断链视为门控不通过。追溯链信息逐阶段累积：阶段 0 分配 REQ ID → 阶段 3 建立 REQ→TP 映射 → 阶段 4 建立 REQ→TP→TC 矩阵 → 交付时汇总为 `99-可追溯性矩阵.md`
