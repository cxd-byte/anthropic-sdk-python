# Anthropic Python SDK 深度解析与中文文档

本文档旨在为中文开发者提供一份详尽的 Anthropic Python SDK 技术参考，不仅包含基础用法，还深入剖析其内部构造与核心技术。

## 1. 模块全解析

本章节将详细拆解 `src/anthropic` 目录下的每一个核心模块，阐述其设计思路与功能。

### 1.1. 客户端层 (`_client.py`, `_base_client.py`)

- **`_client.py`**: 定义了 SDK 的两个主要入口类 `Anthropic` (同步) 和 `AsyncAnthropic` (异步)。这两个类继承自 `_base_client.py` 中的基类，并实现了用户友好的接口。它们负责处理 API Key、认证头、默认查询参数等，并动态挂载了 `completions`、`messages` 等资源访问器。
- **`_base_client.py`**: 提供了客户端的核心逻辑，包括底层的 HTTP 请求发送 (基于 `httpx`)、自动重试机制、超时管理以及响应处理。这里是 SDK 网络功能的核心。

### 1.2. 资源层 (`resources/`)

- **`resources/completions.py`**: 包含了与传统文本补全 API (`/v1/complete`) 交互的 `Completions` 和 `AsyncCompletions` 类。**注意：此 API 已被弃用**，建议迁移到功能更强大的 Messages API。
- **`resources/messages/messages.py`**: 定义了 `Messages` 和 `AsyncMessages` 类，用于访问新一代的 Messages API (`/v1/messages`)。这是 SDK 的核心功能模块，支持多轮对话、工具使用、图片输入等高级功能。
- **`resources/models.py`**: 实现了 `Models` 和 `AsyncModels` 类，允许开发者查询指定模型的详细信息 (`retrieve`) 或获取当前所有可用模型的列表 (`list`)。

### 1.3. 数据模型与类型 (`types/`, `_models.py`)

- **`types/`**: 这个目录包含了 SDK 中所有与 API 端点交互的数据结构定义。每个 `.py` 文件通常对应一个 API 的请求参数 (Params) 或响应体 (Response Body) 的模型。例如，`message_create_params.py` 定义了创建消息时的请求参数结构。这种强类型的设计极大地提升了代码的健壮性和可维护性。
- **`_models.py`**: 定义了所有数据模型的基础类 `BaseModel`，它可能基于 Pydantic，用于实现数据的自动校验和序列化。

### 1.4. 异常处理 (`_exceptions.py`)

- **`_exceptions.py`**: 定义了一套层次分明的自定义异常类，全部继承自 `APIError`。这些异常与具体的 HTTP 错误码紧密关联，例如：
    - `NotFoundError` (404)
    - `RateLimitError` (429)
    - `InternalServerError` (5xx)
  这使得开发者可以编写出非常精细和健壮的错误处理逻辑。

### 1.5. 内部工具与辅助模块

- **`_utils/`**: 包含了一系列内部使用的辅助函数，例如日志配置 (`_logs.py`)、数据转换 (`_transform.py`)、兼容性处理 (`_compat.py`) 等。这些模块为 SDK 的核心功能提供了底层支持。
- **`_streaming.py`**: 实现了处理流式响应的核心逻辑，包括 `Stream` 和 `AsyncStream` 类，它们能够处理 Server-Sent Events (SSE) 并将其解析为对应的数据对象。
- **`pagination.py`**: 提供了对分页 API (如 `models.list`) 的支持，能够自动处理分页逻辑，方便开发者遍历所有结果。

---

## 2. 基础用法

### 2.1. 客户端初始化

**同步:**
```python
import anthropic
client = anthropic.Anthropic(api_key="YOUR_API_KEY")
```

**异步:**
```python
import asyncio
import anthropic
async_client = anthropic.AsyncAnthropic(api_key="YOUR_API_KEY")
```

### 2.2. 发送消息 (Messages API)

```python
message = client.messages.create(
    model="claude-3-5-sonnet-20240620",
    max_tokens=1000,
    messages=[
        {"role": "user", "content": "你好, Claude!"}
    ]
)
print(message.content[0].text)
```

---

## 3. 高级用法

### 3.1. 在消息中使用图片

你可以在 `messages` 的 `content` 列表中加入 `image` 类型的内容块。图片数据需要使用 `base64` 编码。

```python
import base64
import httpx

client = anthropic.Anthropic()

image1_url = "https://upload.wikimedia.org/wikipedia/commons/a/a7/Camponotus_flavomarginatus_worker_casent0173602_profile_1.jpg"
image1_media_type = "image/jpeg"
image1_data = base64.b64encode(httpx.get(image1_url).content).decode("utf-8")

message = client.messages.create(
    model="claude-3-5-sonnet-20240620",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": image1_media_type,
                        "data": image1_data,
                    },
                },
                {
                    "type": "text",
                    "text": "请描述一下这张图片里的昆虫。"
                }
            ],
        }
    ],
)
print(message.content[0].text)
```

### 3.2. 定义和使用工具 (Tools)

Messages API 允许你为 Claude 定义一套工具，模型可以根据对话内容决定调用哪个工具并生成相应的参数。

```python
import json
import anthropic

client = anthropic.Anthropic()

def get_weather(city: str):
    """模拟获取天气的函数"""
    if "北京" in city:
        return json.dumps({"city": "北京", "temperature": "25°C", "description": "晴朗"})
    elif "东京" in city:
        return json.dumps({"city": "东京", "temperature": "22°C", "description": "多云"})
    else:
        return json.dumps({"city": city, "temperature": "N/A"})

tools = [
    {
        "name": "get_weather",
        "description": "根据城市名称获取当前的天气信息。",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称, e.g., 北京, 东京"
                }
            },
            "required": ["city"]
        }
    }
]

response = client.messages.create(
    model="claude-3-5-sonnet-20240620",
    max_tokens=1024,
    messages=[{"role": "user", "content": "今天北京的天气怎么样？"}],
    tools=tools,
    tool_choice={"type": "auto"} # 让模型自动决定是否使用工具
)

# 检查模型是否决定使用工具
tool_use = next((block for block in response.content if block.type == 'tool_use'), None)

if tool_use:
    tool_name = tool_use.name
    tool_input = tool_use.input

    print(f"模型决定使用工具: {tool_name}")
    print(f"工具输入: {tool_input}")

    # 执行工具
    tool_result = get_weather(tool_input['city'])

    # 将工具执行结果返回给模型
    second_response = client.messages.create(
        model="claude-3-5-sonnet-20240620",
        max_tokens=1024,
        messages=[
            {"role": "user", "content": "今天北京的天气怎么样？"},
            {"role": "assistant", "content": response.content},
            {
                "role": "user",
                "content": [
                    {
                        "type": "tool_result",
                        "tool_use_id": tool_use.id,
                        "content": tool_result,
                    }
                ]
            }
        ],
        tools=tools,
    )
    print("模型的最终回答:")
    print(second_response.content[0].text)
```

### 3.3. 精确的异常处理

SDK 定义了丰富的异常类型，你可以根据需要捕获特定的错误。

```python
import anthropic

client = anthropic.Anthropic(api_key="INVALID_KEY")

try:
    client.messages.create(
        model="claude-3-5-sonnet-20240620",
        max_tokens=10,
        messages=[{"role": "user", "content": "hi"}]
    )
except anthropic.AuthenticationError as e:
    print(f"认证失败: {e.__class__.__name__}")
    print(f"HTTP Status Code: {e.status_code}")
    print(f"Response Body: {e.body}")
except anthropic.RateLimitError as e:
    print(f"请求频率过高，请稍后再试: {e.__class__.__name__}")
except anthropic.APIError as e:
    print(f"捕获到通用的 API 错误: {e.__class__.__name__}")

```

### 3.4. 高级客户端配置

你可以对客户端进行更详细的配置，例如设置超时、重试次数和 HTTP 代理。

```python
import httpx
import anthropic

client = anthropic.Anthropic(
    api_key="YOUR_API_KEY",
    # 设置请求超时为 20 秒
    timeout=20.0,
    # 设置最大重试次数为 5 次
    max_retries=5,
    # 配置 HTTP 代理
    http_client=httpx.Client(
        proxies="http://my.proxy.server:8080",
        transport=httpx.HTTPTransport(local_address="0.0.0.0"),
    ),
)
```

---

## 4. 核心技术解析

### 4.1. 架构设计：同步与异步

SDK 巧妙地通过代码生成和基类继承，实现了对同步和异步编程模型的双重支持，同时最大化地复用了核心逻辑。

- **`_base_client.py`**: `SyncAPIClient` 和 `AsyncAPIClient` 在这里定义。它们分别封装了同步的 `httpx.Client` 和异步的 `httpx.AsyncClient`。核心的 `_request` 方法在这里实现，处理了请求的构建、发送、错误处理和重试逻辑，但具体的 HTTP 调用是同步还是异步则由子类决定。
- **`_client.py`**: `Anthropic` 继承 `SyncAPIClient`，而 `AsyncAnthropic` 继承 `AsyncAPIClient`。它们共享大部分的初始化逻辑和属性设置，但底层的网络 I/O 却是完全不同的模式。

这种设计使得 SDK 的维护者可以只修改一处核心逻辑（例如重试策略），就能同时应用到同步和异步两个版本的客户端上。

### 4.2. API 请求的生命周期

一次典型的 API 调用（如 `client.messages.create(...)`）在 SDK 内部经历以下流程：

1.  **参数封装**: 在 `resources/messages/messages.py` 中，`create` 方法接收用户传入的参数，并使用 `types/message_create_params.py` 中定义的模型将这些参数转换成一个结构化的对象。
2.  **请求构建**: `create` 方法内部调用 `self._post`，这个方法继承自 `_base_client.py`。在这里，请求体 (body)、查询参数 (query) 和请求头 (headers) 被最终确定。`api_key` 等认证信息会被自动添加到请求头中。
3.  **HTTP 调用**: `_post` 方法最终会调用 `self._request`，后者会使用其内部持有的 `httpx` 客户端实例来实际执行 HTTP POST 请求。
4.  **响应处理**:
    - 如果请求失败（HTTP 状态码为 4xx 或 5xx），`_base_client` 会根据状态码从 `_exceptions.py` 中选择一个最匹配的异常类并抛出。
    - 如果请求成功，响应体 (JSON) 会被解析。
5.  **数据绑定**: 解析后的 JSON 数据会被绑定到 `types/message.py` 中定义的 `Message` 对象上，最终返回给用户。这个过程确保了返回给用户的是一个类型安全、易于操作的对象，而不是原始的字典。

### 4.3. 流式响应 (Streaming) 的实现

流式响应是基于 Server-Sent Events (SSE) 技术实现的。

1.  当用户调用 `stream()` 方法或在 `create()` 中设置 `stream=True` 时，SDK 会向 API 发送一个带有特殊请求头 (`Accept: text/event-stream`) 的请求。
2.  服务器会返回一个持久化的 HTTP 连接，并以 `data: {...}` 的格式持续不断地推送事件。
3.  在 `_streaming.py` 中，`Stream` 和 `AsyncStream` 类负责处理这个响应流。它们会逐行读取响应，解析出 `data:` 字段后的 JSON 内容。
4.  每个 JSON 事件都会被解析成一个特定的事件对象（例如 `MessageStartEvent`, `ContentBlockDeltaEvent`），然后通过 `yield` 返回给调用者。
5.  SDK 还提供了便利的辅助工具 `MessageStreamManager`，它可以将原始的事件流聚合成更易于使用的文本流 (`text_stream`) 或完整的 `Message` 对象快照 (`get_final_message()`)，极大地简化了开发者的使用体验。
