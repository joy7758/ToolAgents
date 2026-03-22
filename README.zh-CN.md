<!-- language-switch:start -->
[English](./ReadMe.md) | [中文](./README.zh-CN.md)
<!-- language-switch:end -->

# 工具代理

ToolAgents 是一个轻量级且灵活的框架，用于使用各种语言模型和 API 创建函数调用智能体。它提供了一个统一的接口，用于集成不同的LLM提供商并无缝执行函数调用。


## 目录

1. [产品特点](#features)
2. [安装](#installation)
3. [使用方法](#usage)
  - [ChatToolAgent简单使用](#ChatToolAgent)
  - [使用不同的提供商](#Different-Providers)
  - [带有用户循环和聊天历史记录的ChatToolAgent](#Use-ChatToolAgent-with-ChatHistory-class)
  - [具有用户循环和聊天历史记录的流式 ChatToolAgent](#Use-Streaming-ChatToolAgent-with-ChatHistory-class)
4. [自定义工具](#custom-tools)
  - [基于 Pydantic 模型的工具](#1-pydantic-model-based-tools)
  - [基于功能的工具](#2-function-based-tools)
  - [OpenAI式功能规格](#3-openai-style-function-specifications)
  - [良好的文档字符串和描述的重要性](#the-importance-of-good-docstrings-and-descriptions)
5. [贡献](#contributing)
6. [许可证](#license)

## 特征

- 支持多个 LLM 提供商：
  - 开放人工智能API
  - 人择API
  - 米斯特拉尔API
  - OpenAI 类 API，如 OpenRouter、VLLM、llama-cpp-server
- 易于使用的界面，用于向法学硕士传递函数、Pydantic 模型和工具
- 简化函数调用和结果处理的流程
- 统一的消息格式，可以轻松切换提供商，同时保持相同的聊天记录。

## 安装

```bash
pip install ToolAgents
```

## 用法

### 聊天工具代理

```python
import os

from ToolAgents import ToolRegistry
from ToolAgents.agents import ChatToolAgent
from ToolAgents.messages.chat_message import ChatMessage
from ToolAgents.provider import OpenAIChatAPI
from example_tools import calculator_function_tool, current_datetime_function_tool, get_weather_function_tool

from dotenv import load_dotenv

load_dotenv()

# Official OpenAI API
api = OpenAIChatAPI(api_key=os.getenv("OPENAI_API_KEY"), model="gpt-4o-mini")

# Create the ChatAPIAgent
agent = ChatToolAgent(chat_api=api)
settings = api.get_default_settings()
settings.temperature = 0.45
settings.top_p = 1.0

# Define the tools
tools = [calculator_function_tool, current_datetime_function_tool, get_weather_function_tool]
tool_registry = ToolRegistry()

tool_registry.add_tools(tools)
messages = [
    ChatMessage.create_system_message("You are a helpful assistant with tool calling capabilities. Only reply with a tool call if the function exists in the library provided by the user. Use JSON format to output your function calls. If it doesn't exist, just reply directly in natural language. When you receive a tool call response, use the output to format an answer to the original user question."),
    ChatMessage.create_user_message("Get the weather in London and New York. Calculate 420 x 420 and retrieve the date and time in the format: %Y-%m-%d %H:%M:%S.")
]

result = agent.get_streaming_response(
    messages=messages,
    settings=settings, tool_registry=tool_registry)


for res in result:
    print(res.chunk, end='', flush=True)

```

### 不同的提供商
```python
# Import different providers
from ToolAgents.provider import AnthropicChatAPI, OpenAIChatAPI, GroqChatAPI, MistralChatAPI, CompletionProvider

# Official OpenAI API
api = OpenAIChatAPI(api_key=os.getenv("OPENAI_API_KEY"), model="gpt-4o-mini")

# Local OpenAI like API, like vllm or llama-cpp-server
api = OpenAIChatAPI(api_key="token-abc123", base_url="http://127.0.0.1:8080/v1", model="unsloth/Meta-Llama-3.1-8B-Instruct-bnb-4bit")

# Anthropic API
api = AnthropicChatAPI(api_key=os.getenv("ANTHROPIC_API_KEY"), model="claude-3-5-sonnet-20241022")

# Groq API
api = GroqChatAPI(api_key=os.getenv("GROQ_API_KEY"), model="llama-3.3-70b-versatile")


# Mistral API
api = MistralChatAPI(api_key=os.getenv("MISTRAL_API_KEY"), model="mistral-small-latest")

```

### 将 ChatToolAgent 与 ChatHistory 类一起使用
```python
import os

from ToolAgents import ToolRegistry
from ToolAgents.agents import ChatToolAgent
from ToolAgents.messages import ChatHistory

from ToolAgents.provider import OpenAIChatAPI

from example_tools import calculator_function_tool, current_datetime_function_tool, get_weather_function_tool

from dotenv import load_dotenv

load_dotenv()

# Openrouter API
api = OpenAIChatAPI(api_key=os.getenv("OPENROUTER_API_KEY"), model="google/gemini-2.0-pro-exp-02-05:free", base_url="https://openrouter.ai/api/v1")

# Create the ChatAPIAgent
agent = ChatToolAgent(chat_api=api)

# Create a samplings settings object
settings = api.get_default_settings()

# Set sampling settings
settings.temperature = 0.45
settings.top_p = 1.0

# Define the tools
tools = [calculator_function_tool, current_datetime_function_tool, get_weather_function_tool]
tool_registry = ToolRegistry()

tool_registry.add_tools(tools)

chat_history = ChatHistory()
chat_history.add_system_message("You are a helpful assistant with tool calling capabilities. Only reply with a tool call if the function exists in the library provided by the user. Use JSON format to output your function calls. If it doesn't exist, just reply directly in natural language. When you receive a tool call response, use the output to format an answer to the original user question.")

while True:
    user_input = input("User input >")
    if user_input == "quit":
        break
    elif user_input == "save":
        chat_history.save_to_json("example_chat_history.json")
    elif user_input == "load":
        chat_history = ChatHistory.load_from_json("example_chat_history.json")
    else:
        chat_history.add_user_message(user_input)

        chat_response = agent.get_response(
            messages=chat_history.get_messages(),
            settings=settings, tool_registry=tool_registry)

        print(chat_response.response.strip())
        chat_history.add_messages(chat_response.messages)

```

### 将 Streaming ChatToolAgent 与 ChatHistory 类结合使用
```python
import os

from ToolAgents import ToolRegistry
from ToolAgents.agents import ChatToolAgent
from ToolAgents.messages import ChatHistory

from ToolAgents.provider import OpenAIChatAPI

from example_tools import calculator_function_tool, current_datetime_function_tool, get_weather_function_tool

from dotenv import load_dotenv

load_dotenv()

# Openrouter API
api = OpenAIChatAPI(api_key=os.getenv("OPENROUTER_API_KEY"), model="google/gemini-2.0-pro-exp-02-05:free", base_url="https://openrouter.ai/api/v1")

# Create the ChatAPIAgent
agent = ChatToolAgent(chat_api=api)

# Create a samplings settings object
settings = api.get_default_settings()

# Set sampling settings
settings.temperature = 0.45
settings.top_p = 1.0

# Define the tools
tools = [calculator_function_tool, current_datetime_function_tool, get_weather_function_tool]
tool_registry = ToolRegistry()

tool_registry.add_tools(tools)

chat_history = ChatHistory()
chat_history.add_system_message("You are a helpful assistant with tool calling capabilities. Only reply with a tool call if the function exists in the library provided by the user. Use JSON format to output your function calls. If it doesn't exist, just reply directly in natural language. When you receive a tool call response, use the output to format an answer to the original user question.")

while True:
    user_input = input("User input >")
    if user_input == "quit":
        break
    elif user_input == "save":
        chat_history.save_to_json("example_chat_history.json")
    elif user_input == "load":
        chat_history = ChatHistory.load_from_json("example_chat_history.json")
    else:
        chat_history.add_user_message(user_input)

        stream = agent.get_streaming_response(
            messages=chat_history.get_messages(),
            settings=settings, tool_registry=tool_registry)
        chat_response = None
        for res in stream:
            print(res.chunk, end='', flush=True)
            if res.finished:
              chat_response = res.finished_response
        if chat_response is not None:
            chat_history.add_messages(chat_response.messages)
        else:
          raise Exception("Error during response generation")
```
## 定制工具

ToolAgents 支持多种创建自定义工具的方式，允许您将特定功能集成到代理中。以下是创建自定义工具的不同方法：

### 1. 基于 Pydantic 模型的工具

您可以使用 Pydantic 模型创建工具，该模型提供强类型和自动验证。这是计算器工具的示例：

```python
from enum import Enum
from typing import Union
from pydantic import BaseModel, Field
from ToolAgents import FunctionTool

class MathOperation(Enum):
    ADD = "add"
    SUBTRACT = "subtract"
    MULTIPLY = "multiply"
    DIVIDE = "divide"

class Calculator(BaseModel):
    """
    Perform a math operation on two numbers.
    """
    number_one: Union[int, float] = Field(..., description="First number.")
    operation: MathOperation = Field(..., description="Math operation to perform.")
    number_two: Union[int, float] = Field(..., description="Second number.")

    def run(self):
        if self.operation == MathOperation.ADD:
            return self.number_one + self.number_two
        elif self.operation == MathOperation.SUBTRACT:
            return self.number_one - self.number_two
        elif self.operation == MathOperation.MULTIPLY:
            return self.number_one * self.number_two
        elif self.operation == MathOperation.DIVIDE:
            return self.number_one / self.number_two
        else:
            raise ValueError("Unknown operation.")

calculator_tool = FunctionTool(Calculator)
```

### 2. 基于功能的工具

您还可以通过简单的 Python 函数创建工具。这是日期时间工具的示例：

```python
import datetime
from ToolAgents import FunctionTool

def get_current_datetime(output_format: str = '%Y-%m-%d %H:%M:%S'):
    """
    Get the current date and time in the given format.

    Args:
        output_format: formatting string for the date and time, defaults to '%Y-%m-%d %H:%M:%S'
    """
    return datetime.datetime.now().strftime(output_format)

current_datetime_tool = FunctionTool(get_current_datetime)
```

### 3. OpenAI风格的功能规范

ToolAgents 支持根据 OpenAI 风格的功能规范创建工具：

```python
from ToolAgents import FunctionTool

def get_current_weather(location, unit):
    """Get the current weather in a given location"""
    # Implementation details...

open_ai_tool_spec = {
    "type": "function",
    "function": {
        "name": "get_current_weather",
        "description": "Get the current weather in a given location",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "The city and state, e.g. San Francisco, CA",
                },
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
            },
            "required": ["location", "unit"],
        },
    },
}

weather_tool = FunctionTool.from_openai_tool(open_ai_tool_spec, get_current_weather)
```

### 良好的文档字符串和描述的重要性

创建自定义工具时，提供清晰、全面的文档字符串和描述至关重要。这就是为什么它们很重要：

1. **人工智能理解**：语言模型使用这些描述来理解每个工具的目的和功能。更好的描述可以带来更准确的工具选择和使用。

2. **参数清晰度**：每个参数的详细描述有助于人工智能了解预期的输入，减少错误并提高生成的调用的质量。

3. **正确使用**：良好的文档字符串指导人工智能如何正确使用该工具，包括输入的任何特定格式或约束。

4. **错误预防**：通过明确说明预期的输入类型和任何限制，您可以在许多潜在错误发生之前防止它们。

以下是一个文档齐全的工具的示例：

```python
from pydantic import BaseModel, Field
from ToolAgents import FunctionTool

class FlightTimes(BaseModel):
    """
    Retrieve flight information between two locations.

    This tool provides estimated flight times, including departure and arrival times,
    for flights between major airports. It uses airport codes for input.
    """

    departure: str = Field(
        ...,
        description="The departure airport code (e.g., 'NYC' for New York)",
        min_length=3,
        max_length=3
    )
    arrival: str = Field(
        ...,
        description="The arrival airport code (e.g., 'LAX' for Los Angeles)",
        min_length=3,
        max_length=3
    )

    def run(self) -> str:
        """
        Retrieve flight information for the given departure and arrival locations.

        Returns:
            str: A JSON string containing flight information including departure time,
                 arrival time, and flight duration. If no flight is found, returns an error message.
        """
        # Implementation details...

get_flight_times_tool = FunctionTool(FlightTimes)
```

在此示例中，文档字符串和字段描述提供了有关该工具的用途、输入要求和预期输出的清晰信息，使人工智能和人类开发人员能够有效地使用该工具。

## 贡献

欢迎向 ToolAgents 做出贡献！请随时提交拉取请求、创建问题或提出改进建议。

## 执照

ToolAgents 是根据 MIT 许可证发布的。有关详细信息，请参阅 [LICENSE](LICENSE) 文件。
