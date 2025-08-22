<!--
 * @Description: 
 * @version: 
 * @Author: QuZhi
 * @Date: 2025-08-08 11:46:49
 * @LastEditors: QuZhi
 * @LastEditTime: 2025-08-11 00:20:53
-->

# 系统名称：本体增强的大模型系统（Ontology Augmented **Generation** LLM System）

## 系统介绍

检索增强的大模型系统（Retrieval-Augmented Generation LLM System）是现阶段工业界的大模型系统的最佳实践，但是其在实践过程中，仍然存在较多问题，尤其严重的是幻觉问题。这些问题严重的限制了大模型在工业级应用的影响力。

因此，我们提出了本体增强的大模型系统（Ontology Augmented Generation LLM System)。这套系统使用本体（融合了数据定义、数据实例、处理函数等集合概念）作为增强大模型的素材，极大的提高了大模型系统的泛化能力和稳定性。

# 专用名词介绍

### 本体

本体包含了数据定义、数据实例和对其进行操作的处理函数。以下为一个具体的示例。

本项目中，数据定义和数据实例使用excel作为载体，表头第一行定义字段名，表头第二行为数据类型定义，表头第三行为字段自然语言描述解释。

#### 示例：人员信息本体（Person Ontology）

##### 1. 数据定义（类似 Excel 的字段结构）

| 字段（Class/Property） | 数据类型 | 描述           |
| ---------------------- | -------- | -------------- |
| 姓名（name）           | string   | 人员的全名     |
| 年龄（age）            | integer  | 人员的年龄     |
| 性别（gender）         | string   | “男”或“女” |
| 所属部门（department） | string   | 工作部门名称   |
| 入职年份（entry_year） | integer  | 入职公司年份   |

---

##### 2. 数据实例（类似 Excel 表格的每一行数据）

| name | age | gender | department | entry_year |
| ---- | --- | ------ | ---------- | ---------- |
| 张三 | 34  | 男     | 技术部     | 2015       |
| 李四 | 29  | 女     | 市场部     | 2019       |
| 王五 | 42  | 男     | 财务部     | 2010       |

---

##### 3. 处理函数（类似 Excel 的自定义函数或数据分析逻辑）

| 函数名称                | 输入参数         | 输出     | 描述                                                  |
| ----------------------- | ---------------- | -------- | ----------------------------------------------------- |
| `get_avg_age()`       | 无               | 浮点数   | 计算所有人员的平均年龄                                |
| `filter_by_dept(d)`   | 部门名（string） | 人员列表 | 获取指定部门的所有人员                                |
| `years_of_service(n)` | 姓名（string）   | 整数     | 计算该员工入职至今的工作年限（当前年减去 entry_year） |

###### 示例代码（Python 伪代码）

```python
def get_avg_age(data):
    return sum(p["age"] for p in data) / len(data)

def filter_by_dept(data, dept):
    return [p for p in data if p["department"] == dept]

def years_of_service(data, name, current_year=2025):
    p = next(p for p in data if p["name"] == name)
    return current_year - p["entry_year"]

```

# 系统主要外部依赖库及处理方式

## 外部依赖库

### openai-agents-python

#### 安装方式

```shell
pip install openai-agents
```

#### git仓库

git地址：https://github.com/openai/openai-agents-python

###### 基础调用示例

```python

import asyncio
from openai import AsyncOpenAI
from agents.extensions.models.litellm_model import LitellmModel
from agents import Agent, Runner, function_tool, OpenAIChatCompletionsModel, set_default_openai_client, set_default_openai_api, set_tracing_disabled




client = AsyncOpenAI(
    base_url="https://yunwu.ai/v1",
    api_key="sk-w9f6zB53CvsRurhX8wqxyJQUwsQht4m9OV4HNf1Su7o01J9i"
)
set_default_openai_client(client)
set_default_openai_api('chat_completions')
set_tracing_disabled(True)

@function_tool
def get_weather(city: str) -> str:
    #print(f"[debug] getting weather for {city}")
    #pdb.set_trace()
    return f"The weather in {city} is sunny."


async def main(model: str, api_key: str, base_url: str, client: AsyncOpenAI):
    agent = Agent(
        name="Assistant",
        instructions="You only respond in haikus.",
        model=OpenAIChatCompletionsModel(model="o3-mini", openai_client=client),
        tools=[get_weather],
    )

    result = await Runner.run(agent, "What's the weather in Tokyo?")
    print(result.final_output)


if __name__ == "__main__":

    model = "openai/o3-mini"
    api_key = "sk-w9f6zB53CvsRurhX8wqxyJQUwsQht4m9OV4HNf1Su7o01J9i"
    base_url = "https://yunwu.ai/v1"

    asyncio.run(main(model, api_key, base_url, client))
```

#### 说明文档仓库

##### Agent 说明文档

git地址：https://github.com/openai/openai-agents-python/blob/main/docs/agents.md

其中，

###### function tools代码示例

```python
import json

from typing_extensions import TypedDict, Any

from agents import Agent, FunctionTool, RunContextWrapper, function_tool


class Location(TypedDict):
    lat: float
    long: float

@function_tool  # (1)!
async def fetch_weather(location: Location) -> str:
    # (2)!
    """Fetch the weather for a given location.

    Args:
        location: The location to fetch the weather for.
    """
    # In real life, we'd fetch the weather from a weather API
    return "sunny"


@function_tool(name_override="fetch_data")  # (3)!
def read_file(ctx: RunContextWrapper[Any], path: str, directory: str | None = None) -> str:
    """Read the contents of a file.

    Args:
        path: The path to the file to read.
        directory: The directory to read the file from.
    """
    # In real life, we'd read the file from the file system
    return "<file contents>"


agent = Agent(
    name="Assistant",
    tools=[fetch_weather, read_file],  # (4)!
)

for tool in agent.tools:
    if isinstance(tool, FunctionTool):
        print(tool.name)
        print(tool.description)
        print(json.dumps(tool.params_json_schema, indent=2))
        print()
```

###### Agent as tools代码示例

```python
from agents import Agent, Runner
import asyncio

spanish_agent = Agent(
    name="Spanish agent",
    instructions="You translate the user's message to Spanish",
)

french_agent = Agent(
    name="French agent",
    instructions="You translate the user's message to French",
)

orchestrator_agent = Agent(
    name="orchestrator_agent",
    instructions=(
        "You are a translation agent. You use the tools given to you to translate."
        "If asked for multiple translations, you call the relevant tools."
    ),
    tools=[
        spanish_agent.as_tool(
            tool_name="translate_to_spanish",
            tool_description="Translate the user's message to Spanish",
        ),
        french_agent.as_tool(
            tool_name="translate_to_french",
            tool_description="Translate the user's message to French",
        ),
    ],
)

async def main():
    result = await Runner.run(orchestrator_agent, input="Say 'Hello, how are you?' in Spanish.")
    print(result.final_output)
```

##### Handoffs 说明文档

git地址：https://github.com/openai/openai-agents-python/blob/main/docs/handoffs.md

###### handoffs代码示例

```python
from agents import Agent, handoff

billing_agent = Agent(name="Billing agent")
refund_agent = Agent(name="Refund agent")

# (1)!
triage_agent = Agent(name="Triage agent", handoffs=[billing_agent, handoff(refund_agent)])
```

## 外部依赖库处理方式

1.通过git clone对应仓库到项目目录下./3rd_party文件夹中，作为理解代码的参考

2.阅读说明文档，理解调用逻辑

3.pip安装对应库，作为调试基础

# 系统需求

使用上述外部依赖库作为底层基础，在上层构建本体增强的大模型系统（Ontology Augmented **Generation** LLM System）。

这个系统是用python语言编写的一个sdk，不需要前端。

这个系统在conda下的openai_agent_oag环境运行，如果不存在，则创建。

python版本选择3.12。

## 系统输入

1. 用户想要执行的文本指令
2. 得到指令结果所需的多个本体结构，其中每个本体结构包括如下：

   1. 数据定义、数据实例通过excel表格中的sheet传入，读入本体对象对应字段
   2. 面向这个数据结构的分析函数，这些通过用户定义，可以注册到这个本体结构的类上
3. 通用的本体数据分析函数，这些为系统预置，默认输入。具体的例子如：

   1. 随机抽取k条数据
   2. 获取某字段枚举
   3. 获取某字段值范围
   4. 获取某个字段的量化数值分布
   5. 获取两列的相关性
   6. 根据某些列进行数据过滤的工具
4. 选择的基础模型配置（通过配置文件传入），支持openai_compatible的外部调用接口，配置包括：

   1. base_url : end_point节点链接，为：https://yunwu.ai/v1
   2. api_key : 具体访问的api_key，为：sk-w9f6zB53CvsRurhX8wqxyJQUwsQht4m9OV4HNf1Su7o01J9i
   3. model_name : 具体使用的模型名称，为：gpt-4o、o3、o4-mini等

## 系统输出

1. 用户问题对应的答案

## 系统逻辑

### 每一个本体为一个类

入参为excel表格路径、sheet名称、sheet的语义化描述、sheet列名/列数据类型/列语义化描述、注册到这个本体上的专用分析函数。

本体类中包含一个Agent，Agent初始化时，可调用的工具为本体通用/专用分析函数（这些工具需要注册到类中，包装成Function tools）。这个Agent的作用是分析这个本体。

这个Agent的name为“分析xxx本体Agent"。

### 主调度逻辑

#### 第一步，任务拆解

需要初始化一个Agent（命名为agent_plan），把各本体中的Agent作为tools接入。完成对用户任务进行拆解的目标。其拆解结果包括：

1. 各子任务的描述和要达成的目标
2. 各子任务可能需要使用的本体中的Agent
3. 各子任务上一步输出和子任务输入衔接的逻辑
4. 各子任务输出和下一步输入衔接的逻辑

#### 第二步，子任务执行

根据第一步中拆解的结果，顺序执行各子任务，如果子任务执行的实际结果和其与下一步衔接的逻辑有偏离，则重新规划下面的路径。

其中单子任务执行逻辑为：

1. 初始化新的Agent（命名为agent_subtask），把这个子任务在拆解结果中的，需要用到的本体Agent作为handoffs加入agent_subtask。
2. 按照拆解结果中的任务目标，执行agent_subtask
3. 输出结果

其中如果执行结果与预期有偏离，则跳到第一步，根据这步执行结果，重新规划后面的子任务。

#### 第三步，上下文压缩

Agent执行起来上下文可能比较长，所以需要一套上下文压缩机制，在上下文超过max_token。面向任务，保留历史上下文核心信息。

### 其他要求

你需要根据上述逻辑，拆解模块放到目录下对应包的文件夹中，尽可能逻辑清晰、组织有序。

完整实现各模块的功能，不能简化功能模块，更不能留有未实现或者透传参数的逻辑。

## 单元测试和集成测试

### 测试环境

这个系统的测试在conda下的openai_agent_oag环境运行，如果不存在，则创建。

python版本选择3.12。

### 单元测试

你需要把拆解后的模块，都做单元测试。

单元测试文件放到./test/unit_test文件夹中，依照模块组织目录结构，在对应的目录下，准备单元测试脚本和测试数据。

其中测试数据构造时，表头和其中数据一定要列对齐。

### 集成测试

对最终的系统，你需要做集成测试。集成测试的数据和脚本放到./test/intergration_test文件夹中。

其中测试数据构造时，表头和其中数据一定要列对齐。

## 执行样例

需要有一套完整的数据及运行代码示例，数据及脚本放到./example文件夹中。

其中数据构造时，表头和其中数据一定要列对齐。
