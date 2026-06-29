---
name: test-case-reviewer
description: 审查测试用例，检查 P0/P1 测试用例的需求覆盖，验证格式合规性（对照 test-case-format），并提供修订说明。输出门控判定（覆盖度 100% + 格式合规 ≥ 95% + P0/P1 完整）。
license: MIT
compatibility: 需要测试用例和需求文档。
metadata:
  author: sangang
  version: "2.1"
---

# 测试用例评审器 —— 最终质量门

## 触发条件

- 阶段 4 第二步由编排器调用
- 用户直接要求"评审这些测试用例"

## 输入

| 参数 | 来源 | 获取方式 |
|------|------|----------|
| 测试用例 | `{OUTPUT_DIR}/04-test-case-generator-用例.md` | `read_file` |
| 迭代范围 | `{OUTPUT_DIR}/00-迭代范围.md` | `read_file` |
| 测试点报告 | `{OUTPUT_DIR}/03-test-point-analyzer-测试点.md` | `read_file` |
| test-case-format | `skills/test-case-format/SKILL.md` | `read_file`（格式权威来源） |

## 执行步骤

### Step 1：读取所有输入

```python
from hermes_tools import read_file

test_cases = read_file("{OUTPUT_DIR}/04-test-case-generator-用例.md")["content"]
scope = read_file("{OUTPUT_DIR}/00-迭代范围.md")["content"]
tp_report = read_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md")["content"]
format_spec = read_file("skills/test-case-format/SKILL.md")["content"]
```

### Step 2：提取测试用例清单

```python
import re

# 提取所有用例声明
tc_pattern = r"#{2,5}\s+tc-(P\d)：([^\n]+)"
test_cases_list = re.findall(tc_pattern, test_cases)

# 按优先级分组
p0_cases = [(lvl, title) for lvl, title in test_cases_list if lvl == "P0"]
p1_cases = [(lvl, title) for lvl, title in test_cases_list if lvl == "P1"]
p2_cases = [(lvl, title) for lvl, title in test_cases_list if lvl == "P2"]
```

### Step 3：需求覆盖分析

从迭代范围中提取所有需求功能点，与测试用例建立映射：

```python
# 方法1：从用例文件的"追溯矩阵"附录中读取映射
# 方法2：如果无追溯矩阵，通过关键词匹配

# 对每个需求功能点：
#   已覆盖 → 有至少一个 P0 或 P1 用例
#   未覆盖 → 没有用例
#   弱覆盖 → 只有 P2 用例（关键功能必须 P0/P1）
```

### Step 4：追溯链完整性验证（v2.0 新增）

验证 REQ → TP → TC 的全链路完整性：

```python
from hermes_tools import read_file
import re

# 读取所有相关文件
scope = read_file("{OUTPUT_DIR}/00-迭代范围.md")["content"]
tp_report = read_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md")["content"]
test_cases_content = test_cases  # 已读取的用例

# 1. 提取所有 REQ ID
all_reqs = set()
for m in re.finditer(r"REQ-(\d{3})", scope):
    all_reqs.add(f"REQ-{m.group(1)}")

# 2. 提取 REQ→TP 映射（从测试点报告）
req_to_tp = {}  # {REQ-001: [TP001, TP002]}
# 解析测试点报告中的"来源 REQ"字段和"覆盖的 REQ"字段

# 3. 提取 TC→TP 映射（从用例附录）
tc_to_tp = {}  # 从用例文件的附录矩阵中解析

# 4. 逆向构建 REQ→TC 映射
req_to_tc = {}
for req_id in all_reqs:
    tps = req_to_tp.get(req_id, [])
    tcs = []
    for tp in tps:
        tcs.extend(tc_to_tp.get(tp, []))
    req_to_tc[req_id] = tcs

# 5. 验证完整性
traceability_issues = []

for req_id in all_reqs:
    # 检查 1：每条 REQ 都有至少一个 TP
    if req_id not in req_to_tp or len(req_to_tp[req_id]) == 0:
        traceability_issues.append({
            "req": req_id,
            "issue": "REQ → TP 断链",
            "detail": "无对应测试点"
        })
        continue
    
    # 检查 2：每条 REQ 都有至少一个 TC（P0 或 P1）
    tcs = req_to_tc.get(req_id, [])
    if len(tcs) == 0:
        traceability_issues.append({
            "req": req_id,
            "issue": "TP → TC 断链",
            "detail": f"有 TP({req_to_tp[req_id]})但无用例"
        })
        continue
    
    p0_p1_tcs = [tc for tc in tcs if "P0" in tc or "P1" in tc]
    if len(p0_p1_tcs) == 0:
        traceability_issues.append({
            "req": req_id,
            "issue": "缺少 P0/P1 用例",
            "detail": f"仅有 P2 用例: {tcs}"
        })

# 6. 检查反向完整性（无孤儿用例）
# 每个用例都必须追溯到一条 REQ
all_tcs = set()
for tcs in tc_to_tp.values():
    all_tcs.update(tcs)
# 检查是否有用例无法追溯到任何 REQ
```

**追溯链完整判定**：
```python
traceability_chain_ok = len(traceability_issues) == 0
```

### Step 5：格式合规性检查

对照 test-case-format 规范逐项检查：

```python
checks = []

# 检查1：一级标题唯一
h1_count = len(re.findall(r"^# [^#]", test_cases, re.MULTILINE))
checks.append(("一级标题唯一", h1_count == 1, 
    f"{'✅' if h1_count == 1 else '❌ 发现{h1_count}个一级标题'}"))

# 检查2：用例声明格式正确
tc_format_ok = all(
    re.match(r"tc-P\d：", decl) 
    for _, decl in test_cases_list
)
checks.append(("用例声明格式", tc_format_ok))

# 检查3：每个用例有 pc 前置条件
pc_count = len(re.findall(r"##### pc：", test_cases))
checks.append(("前置条件完整", pc_count >= len(test_cases_list),
    f"{pc_count}/{len(test_cases_list)}"))

# 检查4：每个用例步骤后有预期结果
steps = len(re.findall(r"^##### (?!pc：)", test_cases, re.MULTILINE))
results = len(re.findall(r"^###### ", test_cases, re.MULTILINE))
checks.append(("预期结果完整", results >= steps,
    f"步骤{steps}个，预期结果{results}个"))

# 检查5：无空步骤
empty_steps = len(re.findall(r"^#####\s*$", test_cases, re.MULTILINE))
checks.append(("无空步骤", empty_steps == 0,
    f"{'✅' if empty_steps == 0 else '❌ 发现{empty_steps}个空步骤'}"))

# 计算格式合规率
format_pass = sum(1 for _, passed, _ in checks if passed)
format_rate = format_pass / len(checks) * 100
```

### Step 5：P0/P1 完整性检查

```python
# 从需求功能点中识别核心功能（通常标注为"关键"优先级或位于核心业务流程中）
core_features = 识别核心功能点

# 对每个核心功能点：
#   必须有至少一个 P0 用例
#   异常场景必须有 P1 用例覆盖

p0_p1_complete = all(
    功能点有P0覆盖(f) for f in core_features
)
```

### Step 6：质量评估

对每个用例评估可执行性：

| 维度 | 检查标准 |
|------|----------|
| 步骤可执行 | 每步描述具体，可直接操作 |
| 结果可验证 | 预期结果有具体判定标准 |
| 数据具体 | 测试数据有具体取值（非"测试数据"） |
| 场景明确 | 前置条件描述了具体状态（非"正常状态"） |

### Step 8：门控判定并写入门控令牌

```python
from hermes_tools import write_file, read_file
import json

# 计算四个指标
coverage = 需求覆盖度  # 目标: 100%
format_compliance = format_rate  # 目标: ≥ 95%
p0_p1_ok = p0_p1_complete
traceability_ok = len(traceability_issues) == 0  # v2.0 新增：追溯链完整

gate_passed = (
    coverage == 100 
    and format_compliance >= 95 
    and p0_p1_ok
    and traceability_ok
)

# === v2.1 写入门控令牌文件（协议 1） ===
gate_token = {
    "stage": 4,
    "passed": gate_passed,
    "metrics": {
        "coverage": coverage / 100,
        "format_rate": format_compliance / 100,
        "p0_count": len(p0_cases),
        "p1_count": len(p1_cases),
        "p2_count": len(p2_cases),
        "p0p1_complete": p0_p1_ok,
        "traceability_ok": traceability_ok,
        "traceability_issue_count": len(traceability_issues)
    },
    "timestamp": datetime.now().isoformat()
}
write_file("{OUTPUT_DIR}/.gate-4.json", json.dumps(gate_token, ensure_ascii=False, indent=2))

# 强制回读验证
token_readback = read_file("{OUTPUT_DIR}/.gate-4.json")["content"]
assert len(token_readback) > 0, "FATAL: 门控令牌写入失败"
assert '"passed"' in token_readback, "FATAL: 门控令牌缺少 passed 字段"
```

### Step 8：写入评审报告

```python
from hermes_tools import write_file

write_file("{OUTPUT_DIR}/04-test-case-reviewer-评审.md", review_content)
```

## 输出格式

输出文件 `{OUTPUT_DIR}/04-test-case-reviewer-评审.md`：

```markdown
# 测试用例评审报告

**评审日期**: [日期]
**用例总数**: X个（P0: {p0}, P1: {p1}, P2: {p2}）

---

## 一、需求覆盖分析

### 覆盖度

| 指标 | 数值 |
|------|------|
| 需求功能点总数 | {Y} |
| P0/P1 覆盖 | {w} |
| 仅 P2 覆盖（弱覆盖） | {v} |
| 未覆盖 | {u} |
| **覆盖度** | **{w/Y * 100}%** |

### 未覆盖需求

| # | 需求功能点 | 建议 |
|---|-----------|------|
| 1 | {功能点} | 补充 P1 用例 |
| 2 | {功能点} | 补充 P0 用例（核心流程） |

> 如果覆盖度 100%，显示"✅ 所有需求功能点均已有 P0/P1 用例覆盖"

---

## 二、格式合规性检查

### 逐项检查

| # | 检查项 | 结果 | 说明 |
|---|--------|------|------|
| 1 | 一级标题唯一 | {✅/❌} | {说明} |
| 2 | 用例声明格式正确（tc-P0/P1/P2） | {✅/❌} | {说明} |
| 3 | 每个用例有 pc 前置条件 | {✅/❌} | {pc_count}/{total} |
| 4 | 预期结果紧跟步骤 | {✅/❌} | {说明} |
| 5 | 无空步骤 | {✅/❌} | {说明} |

**格式合规率**: {format_rate}%

### 不合规用例清单

| 用例 | 问题 | 建议修订 |
|------|------|----------|
| tc-P1：{标题} | 缺少 pc 前置条件 | 补充前置条件 |
| tc-P0：{标题} | 步骤 3 缺少预期结果 | 补充预期结果 |
| tc-P2：{标题} | 用例声明层级错误（#### → ###） | 修正层级 |

---

## 三、P0/P1 完整性检查

### 核心功能覆盖

| 核心功能 | P0 覆盖 | P1 覆盖 | 状态 |
|----------|---------|---------|------|
| {功能1} | tc-P0-001 ✅ | tc-P1-005 ✅ | ✅ 完整 |
| {功能2} | ✅ | ❌ 缺失异常场景 | ⚠️ 需补充 |
| {功能3} | ❌ | ❌ | ❌ 缺失 |

**P0/P1 完整**: {✅ 是 / ❌ 否}

---

## 四、质量评估

### 可执行性评分

| 维度 | 合规数/总数 | 合规率 |
|------|------------|--------|
| 步骤可执行 | {a}/{total} | {a/total*100}% |
| 结果可验证 | {b}/{total} | {b/total*100}% |
| 数据具体 | {c}/{total} | {c/total*100}% |
| 场景明确 | {d}/{total} | {d/total*100}% |
| **综合可执行性** | | **{avg}%** |

### 需要改进的用例

| 用例 | 问题 | 建议 |
|------|------|------|
| tc-P1：{标题} | 测试数据为"准备测试数据"过于模糊 | 改为"结算金额=1000.00，可用额度=5000.00" |
| tc-P0：{标题} | 前置条件"正常状态"不明确 | 改为"用户已登录，账户状态=正常，可用额度≥1000" |

---

## 五、追溯链完整性（v2.0 新增）

### REQ → TP → TC 断链检查

| 检查项 | 结果 |
|--------|------|
| REQ 总数 | {N} |
| REQ → TP 完整（每条 REQ 有 ≥1 个 TP） | {✅ 是 / ❌ 否} |
| TP → TC 完整（每个 TP 有 ≥1 个 TC） | {✅ 是 / ❌ 否} |
| P0/P1 覆盖（关键功能有高优用例） | {✅ 是 / ❌ 否} |
| 无孤儿用例（每个 TC 可追溯到 REQ） | {✅ 是 / ❌ 否} |
| **追溯链完整** | **{✅ 是 / ❌ 否}** |

### 断链清单（如有）

| REQ ID | 问题 | 详情 |
|--------|------|------|
| REQ-010 | REQ → TP 断链 | 无对应测试点 |
| REQ-015 | TP → TC 断链 | 有 TP020 但无对应用例 |

---

## 六、门控判定（最终质量门）

| 条件 | 当前值 | 阈值 | 结果 |
|------|--------|------|------|
| 需求覆盖度 | {coverage}% | = 100% | {'✅' if coverage == 100 else '❌'} |
| 格式合规率 | {format_rate}% | ≥ 95% | {'✅' if format_rate >= 95 else '❌'} |
| P0/P1 完整 | {'是' if p0_p1_ok else '否'} | 是 | {'✅' if p0_p1_ok else '❌'} |
| 追溯链完整 | {'是' if traceability_ok else '否'} | 是 | {'✅' if traceability_ok else '❌'} |

**最终判定**: {✅ DELIVER / ❌ RETRY}

**下一步**:
- ✅ 交付 → 生成 `README.md` 全链路汇总 + `99-可追溯性矩阵.md`，交付给用户
- ❌ 修订 → 回到 test-case-generator，根据以下修订清单修正：
  1. 未覆盖需求 {u}个 → 补充用例
  2. 格式不合规 {len(格式问题)}个 → 修正格式
  3. P0/P1 缺失 {n}个 → 补充高优用例
  4. 追溯链断链 {len(traceability_issues)}个 → 补充缺失的 TP 或 TC

---

## 六、修订清单（仅 RETRY 时）

| # | 类型 | 用例/需求 | 修订内容 |
|---|------|----------|----------|
| 1 | 覆盖 | {功能点} | 新增 P1 用例 |
| 2 | 格式 | tc-P2：{标题} | 修正预期结果层级 |
| 3 | P0/P1 | {功能点} | 新增 P0 用例覆盖主流程 |
```

## 验证步骤

```python
from hermes_tools import read_file

review = read_file("{OUTPUT_DIR}/04-test-case-reviewer-评审.md")["content"]

# 1. 门控数据自洽
# 2. 每个格式问题都有修订建议
# 3. 修订清单与问题清单一一对应
```

## 门控后续

- **DELIVER** → 编排器生成 `README.md` 全链路汇总
- **RETRY** → 编排器回到 test-case-generator，传入修订清单

## 注意事项

- 覆盖度必须为 **100%**（不是 95%）——这是最终交付门控
- 格式检查必须对照 test-case-format 原文，不凭记忆
- P0/P1 完整性关注核心功能，P2 缺失不阻塞交付
- 修订建议必须具体到"哪个用例的哪个字段需要修改"
