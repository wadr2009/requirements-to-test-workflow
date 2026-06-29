---
name: test-case-generator
description: 从需求文档、测试功能点和后端系统设计生成标准化测试用例，产生可直接导入 XMind 的输出。严格遵循 test-case-format 的 Markdown 层级规范，按场景法组织测试用例。
license: MIT
compatibility: 需要需求文档、测试功能点和后端系统设计。
metadata:
  author: sangang
  version: "2.0"
---

# 测试用例生成器 —— 生成 XMind 可导入的标准化用例

## 触发条件

- 阶段 4 第一步由编排器调用
- 用户直接要求"根据需求生成测试用例"

## ⚠️ 前置强制步骤（必须最先执行）

在生成任何用例之前，**必须先加载格式规范**：

```python
from hermes_tools import read_file

# 1. 加载 test-case-format 格式规范（唯一权威来源）
format_spec = read_file(
    "skills/test-case-format/SKILL.md",
    offset=1,
    limit=200
)["content"]

# 2. 加载其他输入
scope = read_file("{OUTPUT_DIR}/00-迭代范围.md")["content"]
tp_report = read_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md")["content"]
review = read_file("{OUTPUT_DIR}/03-test-feature-reviewer-评审.md")["content"]
# design = read_file("docs/设计文档.md")["content"]  # 如有

# 尝试读取回归范围（如不存在则跳过）
try:
    regression = read_file("{OUTPUT_DIR}/03-regression-scope.md")["content"]
    has_regression = True
except:
    has_regression = False

# 3. 提取格式规则关键点（牢记）：
# - 用例声明: ### tc-P0：标题 或 #### tc-P0：标题（按层级）
# - 前置条件: ##### pc：描述
# - 测试步骤: ##### 操作描述
# - 预期结果: ###### 结果1;结果2;结果3
# - 表单字段验证合并为一个用例
# - 每用例最多 20 步
```

## 输入

| 参数 | 来源 | 获取方式 |
|------|------|----------|
| test-case-format | `skills/test-case-format/SKILL.md` | `read_file`（**必须最先加载**） |
| 迭代范围 | `{OUTPUT_DIR}/00-迭代范围.md` | `read_file` |
| 测试点报告 | `{OUTPUT_DIR}/03-test-point-analyzer-测试点.md` | `read_file` |
| 测试点评审 | `{OUTPUT_DIR}/03-test-feature-reviewer-评审.md` | `read_file` |
| 设计文档（如有） | 用户指定路径 | `read_file` |
| 回归范围（如有） | `{OUTPUT_DIR}/03-regression-scope.md` | `read_file`（如文件存在） |

## 执行步骤

### Step 1：读取所有输入（见上方代码）

### Step 2：按模块组织测试点

从测试点报告中提取所有测试点，按模块分组：

```python
# 解析测试点报告，提取每个测试点的：
# - TP ID
# - 优先级（转为 P0/P1/P2）
# - 所属模块
# - 前置条件
# - 预期结果
# - 依赖关系
```

**优先级映射**：高 → P0，中 → P1，低 → P2

### Step 3：场景法组织测试

对每个测试点，按场景法补充：

1. **主成功场景**（Happy Path）：正常流程，一切顺利
2. **替代场景**（Alternative）：不同路径但同样成功
3. **异常场景**（Exception）：错误输入、系统异常、边界条件

每个场景生成一个独立的测试用例。

### Step 4：生成测试用例内容

对每个用例：

**测试步骤编写规则**：
- 每步一个操作，使用"#####"前缀
- 操作描述要具体：不写"输入数据"，写"在结算金额字段输入 1000.00"
- 预期结果紧跟在操作步骤后，使用"######"前缀
- 多个预期结果用分号分隔

**表单字段验证合并规则**：
- 同一页面的所有字段验证**合并为一个用例**
- 在同一个用例内顺序验证各字段
- 用例标题描述为"XXX页面表单字段验证"

### Step 5：组装为 XMind 格式文件

**严格遵循 test-case-format 的层级规范**：

```markdown
# {ITERATION}迭代测试用例                    ← 一级标题（唯一）

## 一、{功能模块名称}                         ← 二级标题

### 1.{功能分组名称}                          ← 三级标题

#### tc-P0：{测试用例标题}                    ← 四级标题（用例声明）

##### pc：{前置条件描述}                      ← 五级标题（前置条件）

##### {测试步骤1}                             ← 五级标题（操作步骤）
###### {预期结果1};{预期结果2}                ← 六级标题（预期结果）

##### {测试步骤2}
###### {预期结果}
```

**层级规则**：
- 只有一个 `#` 一级标题
- 当功能分组层级较深时，`###` 用例声明可向后延伸至 `####` 或更深
- `##### pc：` 必须跟在用例声明之后
- `######` 预期结果必须直接跟在对应的 `#####` 步骤之后

### Step 6：生成全链路追溯附录

在文件末尾添加附录，记录 REQ → TP → TC 的完整映射：

```markdown
## 附录：全链路追溯矩阵（REQ → TP → TC）

| REQ ID | 需求功能点 | 测试点 | 用例 | 优先级 |
|--------|-----------|--------|------|--------|
| REQ-001 | {功能点描述} | TP001 | tc-P0：{用例标题} | P0 |
| REQ-001 | {功能点描述} | TP002 | tc-P1：{用例标题} | P1 |
```

**生成逻辑**：
1. 从 `{OUTPUT_DIR}/00-迭代范围.md` 提取所有 REQ
2. 从测试点报告提取每个 REQ 对应的 TP
3. 从本文件中的用例反向关联到 TP
4. 确保每条 REQ 至少有 1 行记录（即使未覆盖也要标记）

### Step 6.5：生成回归测试用例（如存在回归范围）

如果 `has_regression=True`，在文件末尾增加回归测试分组：

```python
if has_regression:
    # 解析回归范围报告，提取受影响的 TP
    # 为每个受影响的 TP 生成回归用例
    # 回归用例格式：tc-reg-P0：{原用例标题}【回归-{来源迭代}】
    
    regression_section = """

---

## 回归测试

> 以下用例来自跨迭代回归分析，验证当前迭代变更对历史功能的影响。
> 来源：{OUTPUT_DIR}/03-regression-scope.md

"""
    for item in affected_items:
        regression_section += f"""
### tc-reg-{item['priority']}：{item['tp_desc']}【回归-{item['iteration']}】

##### pc：{item['前置条件']}

##### {回归验证步骤}
###### {预期结果}

"""
    xmind_content += regression_section
```

**回归用例命名规则**：
- 用例声明：`### tc-reg-P0：{原标题}【回归-270】`
- `tc-reg-` 前缀标识回归用例，便于统计和筛选
- `【回归-{来源迭代}】` 后缀标注回归来源

**回归用例优先级**：保持原历史用例优先级，不降级。例如历史 TP 原本 P0 级别，回归用例也是 `tc-reg-P0`。

### Step 7：写入输出

```python
from hermes_tools import write_file

# 输出为一个合并文件
write_file("{OUTPUT_DIR}/04-test-case-generator-用例.md", xmind_content)
```

## 输出格式

输出文件 `{OUTPUT_DIR}/04-test-case-generator-用例.md`：

```markdown
# {ITERATION}迭代测试用例

---

## 一、{功能模块一}

### 1.{功能分组A}

#### tc-P0：{主成功场景标题}

##### pc：{前置条件1}；{前置条件2}

##### {操作步骤1}
###### {预期结果1};{预期结果2}

##### {操作步骤2}
###### {预期结果}

#### tc-P1：{替代场景标题}

##### pc：{前置条件}

##### {操作步骤}
###### {预期结果}

#### tc-P2：{异常场景标题}

##### pc：{前置条件}

##### {操作步骤}
###### {预期结果}

### 2.{功能分组B}

#### tc-P0：{表单字段验证标题}

##### pc：{前置条件}

##### 不填写任何字段，直接点击【提交】
###### 系统显示所有必填字段的错误提示：字段A、字段B、字段C

##### 填写{字段A}为{非法值}，其他字段正常，点击【提交】
###### 系统显示{字段A}的错误提示：{提示语}

##### 填写{字段B}为{边界值}，其他字段正常，点击【提交】
###### {对应的预期结果}

---

## 二、{功能模块二}

...

---

## 附录：全链路追溯矩阵（REQ → TP → TC）

| REQ ID | 需求功能点 | 测试点 | 用例 | 优先级 |
|--------|-----------|--------|------|--------|
| REQ-001 | 工行审核通过后推送开立信息 | TP001 | tc-P0：工行审核通过-开立信息首次推送成功 | P0 |
| REQ-001 | 工行审核通过后推送开立信息 | TP002 | tc-P1：工行审核通过-开立信息补推 | P1 |
| REQ-001 | 工行审核通过后推送开立信息 | TP003 | tc-P1：工行审核通过-超时重试机制 | P1 |
| REQ-002 | 审核驳回后通知申请方 | TP004 | tc-P0：审核驳回-短信通知 | P0 |
| REQ-002 | 审核驳回后通知申请方 | TP005 | tc-P2：审核驳回-站内信通知 | P2 |
| REQ-003 | 额度不足时阻断申请提交 | TP006 | tc-P0：额度不足-阻断申请提交 | P0 |
| ... | ... | ... | ... | ... |

> 此矩阵是生成 `99-可追溯性矩阵.md` 的关键数据源。每条 REQ 都必须至少有一行记录。
```

## 验证步骤

写入后立即自检：

```python
from hermes_tools import read_file
import re

content = read_file("{OUTPUT_DIR}/04-test-case-generator-用例.md")["content"]

# 1. 只有一个一级标题
h1_count = len(re.findall(r"^# [^#]", content, re.MULTILINE))
assert h1_count == 1, f"一级标题必须唯一，当前: {h1_count}"

# 2. 用例声明格式正确（tc-P0/tc-P1/tc-P2）
tc_declarations = re.findall(r"tc-P\d：", content)
assert len(tc_declarations) > 0, "没有找到测试用例声明"

# 3. 每个用例都有 pc 前置条件
pc_count = len(re.findall(r"##### pc：", content))
assert pc_count >= len(tc_declarations), \
    f"前置条件数量({pc_count}) < 用例数量({len(tc_declarations)})"

# 4. 预期结果紧跟步骤（###### )
result_count = len(re.findall(r"###### ", content))
assert result_count > 0, "没有找到预期结果"

# 5. 所有测试点都有对应用例（对照测试点报告）
# 从追溯矩阵中验证
tp_report = read_file("{OUTPUT_DIR}/03-test-point-analyzer-测试点.md")["content"]
tp_in_report = set(re.findall(r"TP(\d+)", tp_report))
tp_in_matrix = set(re.findall(r"TP(\d+)", content))
uncovered = tp_in_report - tp_in_matrix
assert len(uncovered) == 0, f"未覆盖的测试点: {uncovered}"
```

## 注意事项

- **必须先加载 test-case-format**，否则格式必错
- **一个合并文件输出**，不按模块拆分
- 表单字段验证**合并为一个用例**，不在同页面内分字段创建多个用例
- 每个测试用例最多 20 步
- 预期结果用分号分隔多个结果
- **不确定的内容标注"需产品/开发确认"**，不猜测
- 用例标题要描述场景，不是描述测试点 ID
- 追溯矩阵放在文件末尾，帮助 reviewer 做覆盖检查
- **并行模式 REQ 前缀**：如果编排器传入了 REQ 前缀（如 `REQ-A`），所有 REQ ID 必须使用该前缀（`REQ-A01`, `REQ-A02`...），TP ID 同样使用模块前缀（`TP-A001`, `TP-A002`...）。用例文件的溯矩阵中 REQ ID 带前缀，合并时由编排器统一替换为全局 ID
