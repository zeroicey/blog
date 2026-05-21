+++
title = 'Python 探索编程中的接口'
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

# 接口如何玩

Python 处理接口的方式经历了一个非常有趣的演变：从纯粹的动态“鸭子类型”，进化到了结合静态类型检查的 Protocol（结构化子类型）。

这不仅保留了 Python 的灵活性，还解决了大型项目中代码难以维护和缺乏 IDE 提示的痛点。我们来一步步拆解。

1. 鸭子类型 (Duck Typing)：灵活但不可控
鸭子类型是 Python 最核心的动态特性之一。它的哲学是：“如果它走起来像鸭子，叫起来像鸭子，那么它就是鸭子。”

在代码中，这意味着我们不关心对象的真实类型，只关心它是否实现了特定的方法。

```python
class Duck:
    def quack(self):
        print("嘎嘎嘎！")

class Dog:
    def quack(self):
        print("汪？但我也会嘎嘎叫！")

# 只要传入的对象有 quack() 方法，这段代码就能跑通
def make_sound(animal):
    animal.quack()

make_sound(Duck())  # 正常
make_sound(Dog())   # 正常，因为 Dog 恰好也有 quack 方法
```

痛点：
当项目变大时，make_sound(animal) 里的 animal 到底需要什么样的方法？IDE 无法为你自动补全，你只能去翻源码。如果传入了一个没有 quack 方法的对象，只有在运行到那一行时才会报错抛出 AttributeError。

2. 引入 Protocol：静态鸭子类型
为了解决上述痛点，Python 3.8 在 typing 模块中引入了 Protocol（基于 PEP 544）。它被称为结构化子类型 (Structural Subtyping)，也就是“带静态类型检查的鸭子类型”。

它的核心逻辑是：你定义一个“协议”，规定类必须具备哪些方法或属性。只要一个类实现了这些方法，类型检查器（如 mypy）就认为它是这个协议的子类，而完全不需要它去显式继承这个协议。

怎么写 Protocol？
定义 Protocol 时，我们通常只写方法签名，函数体用 ... (Ellipsis) 占位。

```python
from typing import Protocol

# 1. 定义一个协议（接口）
class Quacker(Protocol):
    def quack(self) -> None:
        ...

# 2. 具体的类不需要继承 Quacker！
class RealDuck:
    def quack(self) -> None:
        print("嘎嘎嘎！")

class RubberDuck:
    def quack(self) -> None:
        print("吱吱吱！")

class Cat:
    def meow(self) -> None:
        print("喵喵喵！")

# 3. 在函数参数中标注协议类型
def trigger_sound(animal: Quacker) -> None:
    animal.quack()

trigger_sound(RealDuck())   # ✅ mypy 类型检查通过
trigger_sound(RubberDuck()) # ✅ mypy 类型检查通过

# ❌ mypy 会在这里报错："Cat" is incompatible with "Quacker"
# 但如果在运行时强行跑，只有运行到 trigger_sound 内部才会抛错
trigger_sound(Cat())
```
