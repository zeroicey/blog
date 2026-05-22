+++
title = 'Exploring Interfaces in Python Programming'
date = '2026-05-21T01:40:00+08:00'
draft = false
tags = ['Python', 'Interface', 'Protocol', 'Duck Typing', 'Design Patterns', 'Type Hints', 'Software Architecture', 'OOP']
+++

# Why Interfaces Are Needed

As project scale continues to grow, we need to adapt to more and more third-party packages and libraries. At the same time, we need to consider extensibility and replaceability. Replaceability is particularly important because it allows us to easily switch between different providers. Typical examples include:

- **Computer hardware drivers**: For the same functionality like network or audio, we can use different network cards, microphones, and speakers
- **LLM providers**: We can use different large language models in the same service. For example, `Openclaw` can switch between different models
- **Database selection**: All perform CRUD operations, but we can switch between different databases

However, different network cards, speakers, or microphones come from different manufacturers with different internal implementations. Various LLMs (Qwen, ChatGPT, Claude) have different input/output formats. How do we maintain compatibility with all providers?

**The answer is**:
Provide a unified interface for all external providers. Don't worry about the specific external implementation, only focus on the input/output standards.

For LLMs, internally we just need a function that returns content:

```python
def chat(user_input: str) -> reply: str:
    ...
    return reply
```


For different LLM providers, we can write a similar `chat` function for each, keeping the function parameters and return values highly consistent (this is somewhat like function overloading). This way, our business layer only needs to call the `chat` function without worrying about internal implementation. The function internally handles adaptation and conversion of input/output types.

```python
# Convert custom LLMTool to OpenAI-compatible format
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

# How Interfaces Work in Python

Python's approach to interfaces has undergone an interesting evolution: from pure dynamic "duck typing" to combining static type checking with Protocol (structural subtyping).

This not only preserves Python's flexibility but also solves the pain points of difficult-to-maintain code and lack of IDE hints in large projects. Let's break this down step by step.

## 1. Duck Typing: Flexible but Uncontrollable

Duck typing is one of Python's core dynamic features. Its philosophy is: "If it walks like a duck and quacks like a duck, then it is a duck."

In code, this means we don't care about the object's actual type, only whether it implements specific methods.

```python
class Duck:
    def quack(self):
        print("Quack quack quack!")

class Dog:
    def quack(self):
        print("Woof? But I can quack too!")

# As long as the passed object has a quack() method, this code works
def make_sound(animal):
    animal.quack()

make_sound(Duck())  # Works fine
make_sound(Dog())   # Works fine, because Dog happens to have a quack method
```

**Pain points**:
When the project grows larger, what methods does the `animal` parameter in `make_sound(animal)` actually need? IDE cannot provide auto-completion; you can only look through source code. If an object without a `quack` method is passed, it will only throw an `AttributeError` at runtime when that line is executed.

## 2. Introducing Protocol: Static Duck Typing

To solve the above pain points, Python 3.8 introduced Protocol in the `typing` module (based on PEP 544). It's called Structural Subtyping, or "duck typing with static type checking."

Its core logic is: you define a "protocol" that specifies which methods or attributes a class must have. As long as a class implements these methods, the type checker (like mypy) considers it a subclass of this protocol, completely without needing to explicitly inherit from it.

### How to Write Protocol?

When defining a Protocol, we typically only write method signatures, using `...` (Ellipsis) as a placeholder for the function body.

```python
from typing import Protocol

# 1. Define a protocol (interface)
class Quacker(Protocol):
    def quack(self) -> None:
        ...

# 2. Concrete classes don't need to inherit from Quacker!
class RealDuck:
    def quack(self) -> None:
        print("Quack quack quack!")

class RubberDuck:
    def quack(self) -> None:
        print("Squeak squeak squeak!")

class Cat:
    def meow(self) -> None:
        print("Meow meow meow!")

# 3. Annotate function parameters with protocol types
def trigger_sound(animal: Quacker) -> None:
    animal.quack()

trigger_sound(RealDuck())   # ✅ mypy type check passes
trigger_sound(RubberDuck()) # ✅ mypy type check passes

# ❌ mypy will report error here: "Cat" is incompatible with "Quacker"
# But if forced to run at runtime, error only occurs when trigger_sound executes
trigger_sound(Cat())
```