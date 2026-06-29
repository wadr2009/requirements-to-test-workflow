# 并行策略参考

> 当阶段 2 产出 N 个独立模块时，可按模块并行执行阶段 3+4。
> 并行模式下 REQ ID 需要命名空间隔离，避免不同子代理生成冲突的 ID。
>
> 加载时机：阶段 2 完成后，如果模块数 ≥ 5，编排器应 `read_file("references/parallel-strategy.md")` 获取完整并行策略。

## REQ ID 命名空间规则（并行模式）

在并行模式下，阶段 0 的全局 REQ ID 分配不再适用。改为以下三层命名空间：

| 命名空间 | 格式 | 用途 | 示例 |
|----------|------|------|------|
| **模块 REQ** | `REQ-{模块前缀}{序号}` | 模块内部的功能点 | `REQ-A01`, `REQ-A02`, `REQ-B01` |
| **集成 REQ** | `REQ-X{序号}` | 跨模块的集成功能点 | `REQ-X01`（模块A和B的交集） |
| **回归 REQ** | `REQ-R{序号}` | 回归测试项 | `REQ-R01` |

## 并行前准备：分配 REQ 命名空间

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

## 并行执行

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

## 子代理 context 模板（并行模式）

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

## 强制验证指令
1. 完成后写入 {OUTPUT_DIR}/module-{module_name}/.done.json，格式: {"status": "complete", "tc_count": N}
2. 在文件末尾必须包含"## 附录：全链路追溯矩阵（REQ → TP → TC）"
3. 追溯矩阵中每条 REQ 至少对应 1 行
4. 如果无法满足以上任何一条，在 .done.json 中写 "status": "blocked" 并说明原因
"""
```

## 合并步骤（处理 REQ ID 统一）

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

## REQ 命名空间映射文件格式

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

## 子代理注意事项

- **REQ ID 命名空间隔离**：每个子代理使用独有前缀（REQ-A/B/C...），杜绝 ID 冲突
- **TP ID 同样带前缀**：`TP-A001`, `TP-A002`，避免合并时测试点 ID 冲突
- **跨模块集成测试点不丢失**：由专门的集成代理处理，使用 `REQ-X` 前缀
- **合并时统一 REQ ID**：按 `prefix_to_global_mapping` 将所有模块前缀 REQ 替换为全局连续 REQ
- **追溯矩阵必须包含 REQ 前缀**：方便合并时做前缀→全局映射
- **最终合并后仍需通过阶段 4 门控**：包括追溯链完整性检查
