# 本体增强的大模型系统（Ontology Augmented Generation LLM System）PRD

## 1. 背景与目标
检索增强的大模型系统（Retrieval-Augmented Generation LLM System）虽然是当前工业界的大模型最佳实践，但仍存在幻觉等问题，限制了其在工业场景中的作用。因此，本项目提出以本体为核心的数据增强方案，通过融合数据定义、数据实例与处理函数，提升大模型系统的泛化能力与稳定性。

## 2. 术语与示例
### 2.1 本体
本体由数据定义、数据实例及面向该数据结构的处理函数组成。项目中数据定义与实例均以 Excel 作为载体：
- 第 1 行：字段名
- 第 2 行：数据类型
- 第 3 行：字段自然语言描述

#### 2.1.1 人员信息本体（Person Ontology）示例
**数据定义**

| 字段（Class/Property） | 数据类型 | 描述           |
| ---------------------- | -------- | -------------- |
| 姓名（name）           | string   | 人员的全名     |
| 年龄（age）            | integer  | 人员的年龄     |
| 性别（gender）         | string   | “男”或“女” |
| 所属部门（department） | string   | 工作部门名称   |
| 入职年份（entry_year） | integer  | 入职公司年份   |

**数据实例**

| name | age | gender | department | entry_year |
| ---- | --- | ------ | ---------- | ---------- |
| 张三 | 34  | 男     | 技术部     | 2015       |
| 李四 | 29  | 女     | 市场部     | 2019       |
| 王五 | 42  | 男     | 财务部     | 2010       |

**处理函数**

| 函数名称                | 输入参数         | 输出     | 描述                                                  |
| ----------------------- | ---------------- | -------- | ----------------------------------------------------- |
| `get_avg_age()`       | 无               | 浮点数   | 计算所有人员的平均年龄                                |
| `filter_by_dept(d)`   | 部门名（string） | 人员列表 | 获取指定部门的所有人员                                |
| `years_of_service(n)` | 姓名（string）   | 整数     | 计算该员工入职至今的工作年限（当前年减去 entry_year） |

**示例代码**
```python
def get_avg_age(data):
    return sum(p["age"] for p in data) / len(data)

def filter_by_dept(data, dept):
    return [p for p in data if p["department"] == dept]

def years_of_service(data, name, current_year=2025):
    p = next(p for p in data if p["name"] == name)
    return current_year - p["entry_year"]
```

## 3. 系统整体架构
系统采用分层结构：
1. **本体层（Ontology Layer）**：通过类封装 Excel 表格及专用分析函数，每个本体对应一个 `Agent`，命名为“分析xxx本体Agent”。
2. **调度层（Planning & Execution Layer）**：负责任务拆解、子任务执行与上下文压缩。
3. **基础模型层（LLM Layer）**：调用 `openai-agents-python` 作为底层模型接口。
4. **通用工具层（Utility Layer）**：提供系统预置的通用数据分析函数。

### 3.1 模块划分与接口设计
#### 3.1.1 本体类 `Ontology`
- **初始化** `__init__(self, excel_path: str, sheet_name: str, sheet_desc: str, schema: List[FieldMeta], custom_funcs: List[Callable])`
  - `excel_path`：Excel 文件路径
  - `sheet_name`：Sheet 名称
  - `sheet_desc`：Sheet 的语义化描述
  - `schema`：包含字段名、类型、语义描述的元数据列表
  - `custom_funcs`：专用分析函数集合
- **方法**
  - `load_data() -> List[Dict]`：读取 Excel 数据实例
  - `register_function(func: Callable)`：注册专用分析函数
  - `to_agent() -> Agent`：封装通用及专用函数为工具，返回“分析xxx本体Agent”

#### 3.1.2 通用分析工具模块 `ontology_tools`
- `sample_k(data, k)`：随机抽取 k 条数据
- `enumerate_field(data, field)`：获取字段枚举
- `field_range(data, field)`：获取字段值范围
- `field_distribution(data, field)`：获取字段的量化数值分布
- `field_correlation(data, field_x, field_y)`：计算两列相关性
- `filter_by_conditions(data, conditions)`：按条件过滤数据

#### 3.1.3 任务规划模块 `agent_plan`
- 初始化：`Agent(name="agent_plan", tools=[ontology_agents...])`
- 功能：拆解用户任务，输出子任务描述、需调用的本体 Agent 及输入输出衔接逻辑。

#### 3.1.4 子任务执行模块 `agent_subtask`
- 初始化：`Agent(name="agent_subtask", handoffs=[selected_ontology_agents])`
- 功能：依次执行子任务；如结果与下一步衔接逻辑偏离，则回到任务规划重新规划。

#### 3.1.5 上下文压缩模块 `context_manager`
- `compress(history, max_tokens)`：当上下文超出 `max_tokens` 时，保留与任务相关的核心信息。

## 4. 外部依赖及处理方式
### 4.1 依赖库：`openai-agents-python`
- **安装方式**
```shell
pip install openai-agents
```
- **Git 仓库**：<https://github.com/openai/openai-agents-python>
- **基础调用示例**
```python
import asyncio
from openai import AsyncOpenAI
from agents.extensions.models.litellm_model import LitellmModel
from agents import Agent, Runner, function_tool, OpenAIChatCompletionsModel, set_default_openai_client, set_default_openai_api,\
set_tracing_disabled

client = AsyncOpenAI(
    base_url="https://yunwu.ai/v1",
    api_key="sk-w9f6zB53CvsRurhX8wqxyJQUwsQht4m9OV4HNf1Su7o01J9i"
)
set_default_openai_client(client)
set_default_openai_api('chat_completions')
set_tracing_disabled(True)

@function_tool
def get_weather(city: str) -> str:
    return f"The weather in {city} is sunny."

async def main(model: str, api_key: str, base_url: str, client: AsyncOpenAI):
    agent = Agent(
        name="Assistant",
        instructions="You only respond in haikus.",
        model=OpenAIChatCompletionsModel(model="o3-mini", openai_client=client),
        ...
)
```
- **多 Agent handoff 示例**
```python
from agents import Agent, handoff

billing_agent = Agent(name="Billing agent")
refund_agent = Agent(name="Refund agent")

# (1)!
triage_agent = Agent(name="Triage agent", handoffs=[billing_agent, handoff(refund_agent)])
```

- **处理方式**
  1. 通过 `git clone` 对应仓库到 `./3rd_party` 目录，用于参考代码。
  2. 阅读说明文档，理解调用逻辑。
  3. 使用 `pip` 安装库，供调试使用。

## 5. 运行环境
- **语言**：Python 3.12
- **环境管理**：Conda 环境 `openai_agent_oag`（若不存在则创建）

## 6. 系统输入与输出
### 输入
1. 用户文本指令。
2. 多个本体结构：
   - Excel 表格传入数据定义与实例（按表头与数据行对齐）。
   - 用户自定义处理函数，可注册至本体类。
3. 系统预置通用分析函数（见 3.1.2）。
4. 基础模型配置：
   - `base_url`: `https://yunwu.ai/v1`
   - `api_key`: `sk-w9f6zB53CvsRurhX8wqxyJQUwsQht4m9OV4HNf1Su7o01J9i`
   - `model_name`: 如 `gpt-4o`、`o3`、`o4-mini`

### 输出
- 针对用户问题的最终答案。

## 7. 系统流程
### 7.1 本体初始化
- 以 Excel 路径、Sheet 名称、描述、列信息及专用函数初始化 `Ontology`。
- 注册通用与专用函数并封装为工具，生成“分析xxx本体Agent”。

### 7.2 主调度逻辑
1. **任务拆解**：`agent_plan` 调用各本体 Agent 进行任务分析，输出子任务列表、所需本体 Agent 与输入输出衔接方式。
2. **子任务执行**：依照拆解顺序，`agent_subtask` 依次执行子任务并输出结果；若结果偏离预期衔接逻辑，则回到第 1 步重新规划。
3. **上下文压缩**：当上下文超过 `max_tokens` 时，调用 `context_manager` 保留与任务相关的核心信息。

## 8. 目录与模块要求
- 按模块拆分代码至对应包和文件夹，结构清晰有序。
- 所有功能需完整实现，不得保留未实现逻辑或透传参数。

## 9. 测试方案
### 9.1 单元测试
- 位置：`./test/unit_test`，按模块组织目录结构。
- 为每个模块准备单元测试脚本和对齐的测试数据。

### 9.2 集成测试
- 位置：`./test/intergration_test`。
- 提供完整数据及脚本，数据表头与内容需列对齐。

### 9.3 测试环境
- 在 Conda 环境 `openai_agent_oag` 下运行，Python 版本 3.12。

## 10. 使用示例
- 提供完整的数据与运行示例，存放在 `./example` 目录。
- 示例数据表头与内容需列对齐。

## 11. 交付物
- SDK 代码（无前端）。
- 完整文档与示例。
- 单元测试与集成测试。
