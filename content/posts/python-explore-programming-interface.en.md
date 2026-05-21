+++
title = '探索编程中的接口'
date = '2026-05-21T01:40:00+08:00'
draft = false
+++

# 为什么需要接口

随着项目规模不断的增加，越来越多的第三方包，库需要我们去适配，同时需要去考虑可扩展性，可替换性，可替换尤为重要因为这样才能在不同的提供商方便切换，比较典型的例子有：

- 计算机硬件驱动，同样的功能比如网络，声音，我们可以用不同的网卡，麦克风和音响
- 大模型提供商，可以在同一服务使用不同的大模型，比如 `Openclaw` 可以切换不同的模型
- 数据库选型，都是增删改查，但是我们可以使用不同的数据库去切换

但是，不同的网卡，音响或麦克风他们的厂家都不一致，他们的内部运作方式肯定也不一样，各个大模型（Qwen, Chatgpt, Claude）他们的模型的输入输出的格式都有出入，我们该如何去兼容所有厂商呢？

**答案是**:
给所有外部提供商提供统一的接口，不用担心外部具体的实现，只关心出入的标准

对于大模型，程序内部我们只需要一个可以返回内容的函数
```python
def chat(user_input: str) -> reply: str:
    ...
    return reply
```


不同的大模型提供商，我们可以为他们写一个同样的 `chat` 函数，并且函数参数和返回值都高度保持一致(这有点像函数的重载)，这样我们的业务层就只需要调用 `chat` 函数，不需要关注内部实现，然后在函数内部就去适配，去转换输入输出类型
```python
# 将自定义的 LLMTool 转换成 OpenAi 可接受的格式
def _to_openai_tools(
    self, tools: list[LLMTool]
) -> list[ChatCompletionToolUnionParam]:
    openai_tools: list[ChatCompletionToolUnionParam] = []

    for tool in tools:
        function_def: FunctionDefinition = {
            "name": tool.name,
            "description": tool.description,
            "parameters": tool.input_schema,
        }

        openai_tool: ChatCompletionFunctionToolParam = {
            "type": "function",
            "function": function_def,
        }

        openai_tools.append(openai_tool)

    return openai_tools
```
