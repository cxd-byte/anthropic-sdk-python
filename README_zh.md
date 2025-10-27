# Anthropic Python SDK 中文文档

本文档旨在为中文用户提供 Anthropic Python SDK 的详细参考。

## 核心模块

### `anthropic`

- **文件路径:** `src/anthropic/__init__.py`
- **功能:** 作为 SDK 的主入口，对外暴露了所有核心组件，包括客户端、异常类型、数据模型和常量。

### `anthropic._client`

- **文件路径:** `src/anthropic/_client.py`
- **功能:** 定义了 `Anthropic` (同步) 和 `AsyncAnthropic` (异步) 两个客户端类。它们是与 Anthropic API 交互的核心，负责处理身份验证、请求签名和 API 调用。客户端实例提供了访问不同 API 资源的接口，例如 `completions`、`messages` 和 `models`。

### `anthropic.resources.completions`

- **文件路径:** `src/anthropic/resources/completions.py`
- **功能:** 包含了用于与**传统文本补全 API** (`/v1/complete`) 交互的 `Completions` 和 `AsyncCompletions` 类。该 API 已被弃用，建议迁移到更新的 Messages API。

### `anthropic.resources.messages.messages`

- **文件路径:** `src/anthropic/resources/messages/messages.py`
- **功能:** 包含了用于与**新版 Messages API** (`/v1/messages`) 交互的 `Messages` 和 `AsyncMessages` 类。这是推荐使用的 API，支持多轮对话、图像输入和工具使用等高级功能。

### `anthropic.resources.models`

- **文件路径:** `src/anthropic/resources/models.py`
- **功能:** 定义了 `Models` 和 `AsyncModels` 类，用于与模型 API 交互。它允许用户查询特定模型的信息或获取可用模型列表。

---

## 详细使用说明

### 1. 客户端初始化

首先，你需要导入并初始化客户端。SDK 支持同步和异步两种模式。

**同步客户端:**

```python
import anthropic

client = anthropic.Anthropic(
    # 默认为 os.environ.get("ANTHROPIC_API_KEY")
    api_key="YOUR_API_KEY",
)
```

**异步客户端:**

```python
import asyncio
import anthropic

async_client = anthropic.AsyncAnthropic(
    # 默认为 os.environ.get("ANTHROPIC_API_KEY")
    api_key="YOUR_API_KEY",
)

async def main():
    # ... 在这里写你的异步代码 ...
    pass

if __name__ == "__main__":
    asyncio.run(main())
```

### 2. 使用 Messages API (推荐)

Messages API 是与 Claude 交互的首选方式，功能强大且灵活。

**发送单轮对话请求:**

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-3-opus-20240229",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "你好, Claude!"}
    ]
)

print(message.content)
```

**进行多轮对话:**

只需在 `messages` 列表中按顺序传入之前的对话历史即可。

```python
message = client.messages.create(
    model="claude-3-opus-20240229",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "帮我计划一次去东京的旅行。"},
        {"role": "assistant", "content": "好的！你想去几天？有什么特别想去的地方吗？"},
        {"role": "user", "content": "我想去5天，必去的地方有涩谷十字路口和浅草寺。"}
    ]
)
print(message.content)
```

**流式响应 (Streaming):**

如果你需要实时获取模型的输出，可以使用流式响应。

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    max_tokens=1024,
    messages=[{"role": "user", "content": "给我讲个关于程序员的笑话"}],
    model="claude-3-opus-20240229",
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### 3. 使用 Text Completions API (传统 API)

**注意:** 这是一个传统 API，建议使用更新、功能更强大的 Messages API。

```python
import anthropic

client = anthropic.Anthropic()

completion = client.completions.create(
    model="claude-2.1",
    max_tokens_to_sample=300,
    prompt=f"{anthropic.HUMAN_PROMPT} 你好, Claude. {anthropic.AI_PROMPT}",
)
print(completion.completion)
```

### 4. 列出可用模型

你可以获取当前所有可用模型的列表。

```python
import anthropic

client = anthropic.Anthropic()

models = client.models.list()

for model in models.data:
    print(model.id)
```
