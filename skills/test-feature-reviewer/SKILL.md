---
name: test-feature-reviewer
description: 审查测试功能点，确保测试范围完整，验证测试点与需求的一致性。计算覆盖度、识别不一致项、评估可测试性，输出门控判定。
license: MIT
compatibility: 需要需求文档和测试功能点。
metadata:
  author: sangang
  version: "2.1"
---

# 测试点评审器 —— 覆盖度与一致性验证

## 触发条件

- 阶段 3 第二步由编排器调用
- 用户直接要求"评审这些测试功能点"

## 输入

| 参数 | 来源 | 获取方式 |
|------|------|----------|
| 测试点报告 | `{OUTPUT_DIR}/03-test-point-analyzer-测试点.md` | `read_file` |
| 迭代范围 | `{OUTPUT_DIR}/00-迭代范围.md` | `read_file` |
| 模块拆分（参考） | `{OUTPUT_DIR}/02-module-splitter-模块拆分.md` | `read_file` |

## 执行步骤

### Step 1：读取输入

```python
from hermes_tools import read_file, search_files
import re

tp_report = read_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md")["content"]
scope = read_file("{OUTPUT_DIR}/00-迭代范围.md")["content"]
```

### Step 2：提取功能点清单

从需求文档中提取所有功能点，形成"需求功能点清单"：

```python
# 从迭代范围文档中解析功能点
# 每个功能点记为一个独立条目
# 格式：{功能点ID} | {功能点名称} | {来源段落}
```

### Step 3：建立覆盖映射

将每个需求功能点与测试点进行映射：

```python
coverage_map = {}

for req_point in requirement_points:
    matched_tps = []
    for tp in test_points:
        if 测试点描述匹配需求功能点:
            matched_tps.append(tp["id"])
    
    if matched_tps:
        coverage_map[req_point["id"]] = {
            "status": "已覆盖",
            "test_points": matched_tps
        }
    else:
        coverage_map[req_point["id"]] = {
            "status": "未覆盖",
            "test_points": []
        }
```

### Step 4：计算覆盖度

```python
total_req = len(requirement_points)
covered = sum(1 for v in coverage_map.values() if v["status"] == "已覆盖")
coverage_rate = covered / total_req * 100
```

### Step 5：一致性检查

对每个已覆盖的测试点，检查与需求的一致性：

```python
inconsistencies = []

for tp in test_points:
    # 从需求中找原始描述
    req_desc = 对应需求的原始描述
    
    # 对比方向是否一致
    if 测试点描述与需求方向矛盾:
        inconsistencies.append({
            "test_point": tp["id"],
            "issue": "方向矛盾",
            "requirement": req_desc,
            "test_point_desc": tp["description"]
        })
    
    # 检查是否过度测试（对不存在的功能点创建了测试点）
    if 测试点不对应任何需求功能点:
        inconsistencies.append({
            "test_point": tp["id"],
            "issue": "多余测试点（不对应任何需求）",
            "requirement": "无",
            "test_point_desc": tp["description"]
        })
```

### Step 6：可测试性评估

对每个测试点评估可测试性：

| 维度 | 检查标准 |
|------|----------|
| 前置条件明确 | 前置条件具体可操作，非模糊表述 |
| 预期结果可验证 | 预期结果有具体判定标准，非"正常"、"成功"等 |
| 测试数据可构造 | 测试数据有明确取值或构造方法 |
| 无外部依赖阻塞 | 不依赖不可控的外部系统（或已标注 Mock 方案） |

### Step 7：生成评审报告并写入门控令牌

```python
from hermes_tools import write_file, read_file
import json

# 门控判定
gate_passed = coverage_rate >= 95 and len(inconsistencies) == 0

# === v2.1 写入门控令牌文件（协议 1） ===
gate_token = {
    "stage": 3,
    "passed": gate_passed,
    "metrics": {
        "coverage": round(coverage_rate / 100, 2),
        "total_requirement_points": total_req,
        "covered": covered,
        "uncovered": uncovered,
        "inconsistencies": len(inconsistencies)
    },
    "timestamp": datetime.now().isoformat()
}
write_file("{OUTPUT_DIR}/.gate-3.json", json.dumps(gate_token, ensure_ascii=False, indent=2))

# 强制回读验证
token_readback = read_file("{OUTPUT_DIR}/.gate-3.json")["content"]
assert len(token_readback) > 0, "FATAL: 门控令牌写入失败"

write_file("{OUTPUT_DIR}/03-test-feature-reviewer-评审.md", report_content)
```

## 输出格式

输出文件 `{OUTPUT_DIR}/03-test-feature-reviewer-评审.md`：

```markdown
# 测试功能点评审报告

**评审日期**: [日期]
**测试点总数**: X个
**需求功能点总数**: Y个

---

## 一、需求覆盖分析

### 覆盖度

| 指标 | 数值 |
|------|------|
| 需求功能点总数 | {Y} |
| 已覆盖 | {covered} |
| 未覆盖 | {uncovered} |
| **覆盖度** | **{coverage_rate}%** |

### 已覆盖需求（{covered}个）

| # | 需求功能点 | 对应测试点 | 覆盖类型 |
|---|-----------|-----------|----------|
| 1 | {功能点} | TP001, TP002 | 完整覆盖 |
| 2 | {功能点} | TP005 | 部分覆盖（缺少异常场景） |

### 未覆盖需求（{uncovered}个）

| # | 需求功能点 | 缺失原因 | 建议补充测试点 |
|---|-----------|----------|---------------|
| 1 | {功能点} | 边界场景未覆盖 | 补充金额为0的测试点 |
| 2 | {功能点} | 完全遗漏 | 新增反向场景测试点 |

---

## 二、一致性验证

### 不一致项（{len(inconsistencies)}个）

| # | 测试点 | 问题描述 | 需求描述 | 测试点描述 | 建议 |
|---|--------|----------|----------|-----------|------|
| 1 | TP015 | 方向矛盾 | 审核驳回后不可重新提交 | 审核驳回后可重新提交 | 修正测试点方向 |
| 2 | TP030 | 多余测试点 | 无对应需求 | 测试XXX功能 | 删除此测试点 |

> 如无不一致项，显示"✅ 无不一致项"

---

## 三、测试点质量评估

### 高质量测试点

| 测试点 | 亮点 |
|--------|------|
| TP001 | 前置条件明确，预期结果具体可验证 |
| TP010 | 覆盖了并发场景，边界条件考虑周全 |

### 需要改进的测试点

| 测试点 | 问题 | 建议 |
|--------|------|------|
| TP005 | 预期结果模糊："系统正常处理" | 改为具体的状态变更或提示文案 |
| TP012 | 前置条件缺失 | 补充需要的数据状态 |
| TP020 | 缺少测试数据说明 | 补充具体的测试数据取值 |

---

## 四、可测试性评分

| 维度 | 合格数/总数 | 合规率 |
|------|------------|--------|
| 前置条件明确 | {a}/{total} | {a/total*100}% |
| 预期结果可验证 | {b}/{total} | {b/total*100}% |
| 测试数据可构造 | {c}/{total} | {c/total*100}% |
| 无阻塞依赖 | {d}/{total} | {d/total*100}% |
| **综合可测试性** | | **{avg}%** |

---

## 五、门控判定

| 条件 | 当前值 | 阈值 | 结果 |
|------|--------|------|------|
| 覆盖度 | {coverage_rate}% | ≥ 95% | {'✅' if coverage_rate >= 95 else '❌'} |
| 不一致项 | {len(inconsistencies)} | = 0 | {'✅' if len(inconsistencies) == 0 else '❌'} |

**判定结果**: {PASS / RETRY}

**下一步**:
- ✅ 门控通过 → 进入阶段 4（测试用例生成）
- ❌ 门控未通过 → 回到 test-point-analyzer，根据以下问题修正：
  1. 未覆盖需求: {uncovered}个 → 补充测试点
  2. 不一致项: {len(inconsistencies)}个 → 修正或删除

---

## 六、改进建议

1. {建议1}
2. {建议2}
```

## 验证步骤

用 `read_file` 回读评审报告并检查：
- 门控判定与数据一致（高覆盖度必须有 PASS）
- 每个不一致项都有"建议"
- 可测试性评分四个维度都有数据

## 门控后续

- **PASS** → 编排器自动进入阶段 4
- **RETRY** → 编排器回到 test-point-analyzer：
  - 未覆盖需求 → 补充对应测试点
  - 不一致项 → 修正或删除对应测试点
  - 低质量测试点 → 改进前置条件/预期结果

## 注意事项

- 覆盖度必须基于**需求功能点原文**计算，不可凭记忆
- 不一致项检查要逐条对比需求原文和测试点描述，**不可粗判**
- 多余测试点（不对应任何需求）必须标注为不一致项
- 可测试性评分关注的是"能否被执行"，不是"覆盖是否全面"
