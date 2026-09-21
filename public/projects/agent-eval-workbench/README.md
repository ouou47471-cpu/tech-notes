# Agent Eval Workbench：工具调用评测工作台

## 项目一句话

一个可复现的 Agent 评测工作台，把“最终回答看起来是否正确”拆解为工具选择、参数准确性、RAG 引用、拒答、JSON 输出、延迟和 token 七个可追踪维度。

## 项目背景

工具调用 Agent 的错误经常出现在中间层：

- 工具名称选错
- 工具选对但城市或污染物参数错误
- RAG 回答引用了不存在的条款
- 本该拒答时仍然生成答案
- 模型返回无效 JSON
- 新 Prompt 修复一例却导致其他案例回归

只看最终回答无法判断问题发生在哪一步。这个工作台使用固定测试集比较不同 Agent、模型和 Prompt，让问题可以被定位和回归验证。

## 与现有项目的关系

| 项目 | 角色 |
| --- | --- |
| 多城市空气质量监测 Agent | 提供真实 Agent、工具调用和 trace 数据 |
| SFT Audit Kit | 审计训练和评测数据本身的质量 |
| Agent Eval Workbench | 审计 Agent 的工具调用、引用、拒答和运行表现 |

空气质量 Agent 是应用，Eval Workbench 是从它真实评测中抽象出的通用评测工具。

## 评测架构

![Agent Eval Workbench 架构](public/projects/agent-eval-workbench/architecture.png)

输入包含两部分：

- `cases.jsonl`：问题、期望工具、期望参数、引用和拒答标注
- `results.jsonl`：实际工具调用、参数、引用、拒答、JSON、延迟和 token

工具先做归一化，再计算各维度准确率，最后输出 HTML、JSON、CSV 或对比报告。

## 当前功能

- 顺序无关的工具集合比较
- 完全正确、部分正确和错误判定
- 缺失工具和额外工具统计
- 分类准确率和错误分布
- 城市、污染物等参数准确性检查
- RAG 引用完全匹配率和平均召回率
- 低置信拒答准确率
- JSON 结构化输出有效率
- 平均延迟、P50 和 P95
- Prompt 与 completion token 统计
- 规则 Agent 与 LLM Agent 对比
- `--fail-under` 最低准确率门禁

## 输入格式

案例：

```json
{
  "id": "case-001",
  "question": "北京未来24小时PM2.5会超标吗",
  "expected_tools": ["get_forecast"],
  "expected_params": {
    "get_forecast": {
      "city": "beijing",
      "pollutant": "PM25"
    }
  },
  "category": "forecast"
}
```

结果：

```json
{
  "id": "case-001",
  "actual_calls": [
    {
      "tool": "get_forecast",
      "params": {
        "city": "beijing",
        "pollutant": "PM25"
      }
    }
  ],
  "latency_ms": 180,
  "tokens_prompt": 120,
  "tokens_completion": 28,
  "json_valid": true
}
```

## 真实空气质量 Agent 复测

使用原有的 60 条工具选择回归集：

| 模式 | 完全正确 | 准确率 | 失败 |
| --- | ---: | ---: | ---: |
| 规则 Agent | 60/60 | 100% | 0 |
| LangGraph LLM Agent | 58/60 | 96.7% | 2 |

LLM 版剩余两个错误都集中在 `get_alerts` 与 `get_comprehensive_alerts` 的粒度边界。

![规则版与 LLM 版对比](public/projects/agent-eval-workbench/comparison.png)

## 增强能力基准

新增 6 条针对性基准案例，覆盖参数错误、引用缺失、拒答失败和 JSON 无效：

| 指标 | 结果 |
| --- | ---: |
| 工具选择准确率 | 100% |
| 参数准确率 | 75% |
| 引用完全匹配率 | 50% |
| 引用平均召回率 | 75% |
| 拒答准确率 | 50% |
| JSON 有效率 | 83.3% |
| P95 延迟 | 350 ms |

![增强能力评测报告](public/projects/agent-eval-workbench/enhanced-report.png)

这组数据故意包含错误案例，主要用于验证评测器能否准确定位问题，而不是代表最终 Agent 的线上表现。

## 关键问题与解决方案

### 问题 1：只检查工具名，无法发现参数错误

现象：

Agent 调用了正确工具，却把上海传成北京、把 PM2.5 传成 PM10，工具准确率仍然显示正确。

解决方案：

增加工具级参数标注和参数匹配。城市、污染物缩写和大小写先归一化，再逐项比较期望参数与实际参数。

### 问题 2：RAG 回答缺少引用质量指标

现象：

最终答案文字正确，但引用条款不存在、引用不完整，无法通过工具名判断。

解决方案：

增加引用 ID 归一化、完全匹配率和平均召回率。后续可以扩展为条款标题和语义级别的匹配。

### 问题 3：无法判断拒答行为是否正确

现象：

低置信问题本应拒答，但模型仍生成答案；也可能把正常问题错误拒答。

解决方案：

在案例中增加 `expected_refusal`，与实际 `refused` 字段比较，单独输出拒答准确率。

### 问题 4：结构化输出没有校验

现象：

工具选择和参数正确，但模型返回的原始 JSON 已经损坏，实际链路可能无法执行。

解决方案：

优先读取显式 `json_valid`；没有该字段时，对 `raw` 内容执行 JSON 解析，统计结构化输出有效率。

### 问题 5：只有一个总分，无法比较模型和 Prompt

现象：

单次评测只能看到当前结果，不能快速判断新 Prompt 是否真正优于旧版本。

解决方案：

增加 `compare` 命令，将多个结果文件放在同一张表中比较准确率、参数、引用、拒答、JSON 和 P95 延迟。

## 工程决策

- 核心评测不调用模型，不依赖 API Key，保证可重复和可接入 CI。
- 工具集合按集合比较，不依赖调用顺序。
- 参数归一化支持中英文城市名和污染物常见写法。
- 引用先使用 ID 或标题归一化，避免完全依赖生成文本格式。
- 延迟和 token 缺失时显示为空，不用伪造默认值。
- `--fail-under` 只在工具准确率低于阈值时返回失败，适合作为质量门禁。

## 运行方式

Windows 用户可以直接双击：

- `run_demo.bat`：运行规则版、LLM 版、增强基准和方案对比
- `evaluate_agent.bat`：输入自己的 `cases.jsonl` 与 `results.jsonl`

单次评测：

```bash
agent-eval evaluate --cases cases.jsonl --results results.jsonl --output reports/run
```

多方案对比：

```bash
agent-eval compare --cases cases.jsonl --result rule=rule.jsonl --result llm=llm.jsonl --output reports/compare
```

最低准确率门禁：

```bash
agent-eval evaluate --cases cases.jsonl --results results.jsonl --output reports/run --fail-under 0.95
```

## 项目边界

- 参数目前采用规范化后的精确匹配，尚未支持时间范围等复杂结构。
- 引用目前按 ID 或标题匹配，没有判断文字语义是否等价。
- 真实空气质量数据没有完整标注期望参数、引用和拒答，只有增强基准覆盖这些维度。
- 目前需要提供 `results.jsonl`，还没有直接从 trace 数据库生成。

## 下一阶段

- 从空气质量 Agent 的 `trace.db` 自动生成评测结果
- 支持参数部分匹配和多层嵌套结构
- 增加引用语义匹配
- 增加多轮对话与上下文依赖案例
- 记录数据版本和准确率趋势
- GitHub Actions 自动运行回归评测
