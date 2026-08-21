---
title: Python 的三个关键字：async，await，yield
date: 2025-08-31T20:32:19+08:00
categories: 
  - 技术
tags:
  - python
  - 笔记
draft: false
description: "本文介绍了Python中的三个重要关键字async、await和yield，解释了它们的含义、作用以及在异步编程和生成器中的应用，并通过实际案例展示了如何使用这些关键字提高程序性能。"
---

最近新了解到三个 Python 关键字，想来也是颇有妙用。
- `async + await` 可以实现异步编程，做到多任务处理，同时又不会像多线程一样占据很多资源。
- 而 `yield` 则帮助代码实现懒加载功能，只有在调用时，也就是真正使用的时候，才进行计算，避免了大量数据堆积，又或者执行时间过长。

接下来，我会写的内容有：
1. 对 `async`、`await` 和 `yield` 的解释
2. 简单介绍一下异步编程和协程
3. 同时对比同步与异步的差异
4. 给出一个小案例（文件上传）
5. 最后就是对本文做出总结

## `async`、`await` 和 `yield` 各自的含义和作用

### `async`

- **含义**：用于定义异步函数（协程）的关键字，标记该函数内部可能包含异步操作，需要通过事件循环来调度执行。
- **作用**：声明一个函数为协程，使其能够在内部使用 `await` 关键字。
- **语法**：在函数定义前加上 `async` 关键字，例如 `async def funcname()`。
```python
# 定义异步函数（协程）
async def async_function():
    print("这是一个异步函数")
    # 可以在内部使用await调用其他可等待对象
    await another_async_operation()
    print("执行结束")
```

### `await`

- **含义**：用于暂停当前协程的执行，等待另一个可等待对象（如协程、Task、Future 等）完成，并可以获取其返回结果。
- **作用**：在等待期间主动让出 CPU 执行权，让事件循环可以调度其他就绪的任务，实现非阻塞等待，提高程序效率。
- **语法**：
```python
import asyncio

async def fetch_data():
    # 模拟异步请求
    await asyncio.sleep(2)  # 无阻塞等待 2 秒
    return "返回的数据"

async def main():
    # 等待 fetch_data 完成工作，并获取返回值
    data = await fetch_data()
    print(data)  # 输出: 返回的数据

asyncio.run(main()) # 运行协程
```

### `yield`

- **含义**：在生成器函数中使用，用于暂停函数执行并返回一个值，下次调用生成器时会从暂停的位置继续执行。
- **作用**：将函数转换为生成器，实现迭代器的功能，可按需生成数据（惰性计算），节省内存。
- **语法**：
```python
# 定义生成器函数
def number_generator(n):
    for i in range(n):
        yield i  # 每次调用next()时返回i，并暂停执行
        print(f"生成{i}后继续")

# 创建生成器对象
gen = number_generator(3)

# 迭代生成器
print(next(gen))  # 输出：0
print(next(gen))  # 输出：生成0后继续 → 1
print(next(gen))  # 输出：生成1后继续 → 2
print(next(gen))  # 触发StopIteration异常（生成结束）
```

#### `yield` 与 `return` 的区别

`yield` 和 `return` 都是 Python 中用于在函数中返回值的关键字，但它们的工作机制和使用场景有本质区别，核心差异在于 **是否终止函数执行** 和 **返回值的使用方式**。

- `return`：终止函数并返回结果
	- **作用**：立即终止函数执行，将结果返回给调用者，函数后续代码不再执行。
	- **特点**：一个函数中可以有多个 `return`，会根据条件执行其中一个（因为执行后函数就结束了）。
	- **适用场景**：普通函数，用于返回计算结果并结束函数。
	- **示例**：
```python
def add(a, b):
    return a + b  # 返回结果后，函数立即终止
    print("这里不会执行")  # 此处永远不会执行

result = add(2, 3)
print(result)  # 输出: 5
```

- `yield`：暂停函数并返回当前结果
	- **作用**：暂停函数执行，返回一个当前结果给调用者，但函数不会终止，下次调用时会从暂停处继续执行。
	- **特点**：包含 `yield` 的函数会变成 **生成器（generator）**，调用时不会立即执行，而是返回一个生成器对象，需要通过 `next()` 或迭代来触发执行。
	- **适用场景**：需要“按需生成”数据的场景（如大数据迭代、惰性计算），避免一次性生成所有数据占用大量内存。
	- **示例**：
```python
def number_generator(n):
    for i in range(n):
        yield i  # 暂停函数，返回i，下次从这里继续
        print(f"生成{i}后继续执行")  # 下次调用时会执行这里

# 调用生成器函数，返回生成器对象（不立即执行）
gen = number_generator(3)

# 第一次调用 next (): 执行 yield 0、返回 0 并暂停
print(next(gen))  # 输出: 0

# 第二次调用 next (): 从暂停继续，输出 “生成 0 后续继续执行”, 然后 yield 1, 返回 1
print(next(gen))  # 输出: 生成0后继续执行 → 1

# 第三次调用 next (): 同样，返回 2
print(next(gen))  # 输出: 生成1后继续执行 → 2

# 第四次调用 next (): 循环结束，触发 StopIteration 异常
print(next(gen))  # 触发器停止迭代异常
```

总结就是：
- `return` 是 “一去不返”：返回结果后函数彻底结束。
- `yield` 是 “藕断丝连”：返回中间结果后暂停，下次可从暂停处继续执行。

## 什么是异步编程

异步编程是一种编程模式，主要解决程序中“等待”操作（如网络请求、文件读写等）导致的效率问题，让程序在等待时不阻塞，继续执行其他任务。

### 核心问题：同步编程的痛点

同步编程中，任务按顺序执行，遇到耗时操作（比如请求网络数据）时，程序会一直“卡住”等待结果，期间无法做其他事，导致效率低下。

例如：
```python
import time

# 同步方式
def get_data():
    # 模拟网络请求（耗时2秒）
    time.sleep(2)
    return "数据"

print("开始")
data = get_data()  # 这里会卡住2秒，只能等待
print("数据拿到了：", data)
print("结束")
```
这段代码会在 `get_data()` 处等待 2 秒，期间无法执行后续操作。

### 异步编程的解决思路

异步编程允许程序在等待耗时操作时，暂时“跳过”这个任务，去执行其他任务，等耗时操作完成后再回来处理结果。

关键概念：
- **事件循环（Event Loop）**：相当于“调度中心”，负责管理所有任务的执行顺序，决定什么时候执行哪个任务。
- **协程（Coroutine）**：特殊的函数（用 `async def` 定义），可以被暂停和恢复，是异步任务的基本单位。
- **await**：在协程中使用，当遇到耗时操作（如网络请求）时，用 `await` 告诉事件循环：“我要等结果，先去执行别的任务吧”。

### 如何在 Python 中使用异步编程

在 Python 中使用异步编程主要依赖 `asyncio` 库，核心是通过 `async/await` 语法定义和调度协程。以下是具体步骤和示例：

#### 运行协程

要运行一个协程，可以使用 `asyncio.run()` 函数。它会创建一个事件循环，并运行指定的协程。

```python
import asyncio

async def main():
    print("开始")
    await asyncio.sleep(3)  # 暂停3秒，模拟耗时长的操作
    print("结束")

asyncio.run(main())
```

#### 并发执行

可以使用 `asyncio.gather()` 函数并发执行多个协程，并等待它们全部完成。

```python
import asyncio

async def task1():
    print("开始任务：1")
    # 模拟 I/O 操作 (非阻塞等待)
    await asyncio.sleep(1)
    print("任务完成：1")

async def task2():
    print("开始任务：2")
    # 模拟 I/O 操作 (非阻塞等待)
    await asyncio.sleep(2)
    print("完成任务：2")

async def main():
    await asyncio.gather(task1(), task2())

asyncio.run(main())
```

#### 创建任务

任务是对协程的封装，表示一个正在执行或将要执行的协程。可以通过 `asyncio.create_task()` 函数创建任务，创建后会被加入事件循环的任务队列，自动开始运行（除非事件循环未启动）。

```python
async def say_hello():
    print("Hello!")

async def main():
    task = asyncio.create_task(say_hello())
    await task
```

而 `asyncio.gather()` 会自动将协程包装为 Task 并启动（等价于先调用 `create_task()`），同时也可以传入 `asyncio.create_task()` 创建的任务。

当显式调用 `asyncio.create_task()` 时，我们可以更灵活的创建，比如说为协程传入指定的参数、使用循环批量生成任务。

#### 超时控制

使用 `asyncio.wait_for()` 函数为协程设置超时时间。如果协程在指定时间内未完成，将引发 `asyncio.TimeoutError` 异常。

```python
import asyncio

async def long_task():
    await asyncio.sleep(10)
    print("Task finished")

async def main():
    try:
        # 限制任务必须在5秒内完成
        await asyncio.wait_for(long_task(), timeout=5)
    except asyncio.TimeoutError:
        print("Task timed out")

asyncio.run(main())  # 输出：Task timed out
```

### 异步编程示例

``python
import asyncio

# 定义异步任务（协程）
async def get_data():
    print("开始请求数据...")
    await asyncio.sleep(2) # 模拟网络请求 (非阻塞等待)
    print("数据请求完成")
    return "数据"

async def other_task():
    print("执行其他任务...")
    await asyncio.sleep(1) # 模拟网络请求 (非阻塞等待)
    print("其他任务完成")

# 主协程
async def main():
    # 同时启动两个任务
    await asyncio.gather(get_data(), other_task())

# 启动事件循环
asyncio.run(main())
```

**执行顺序**：
1. 事件循环启动，执行 `main()`，创建两个任务并开始运行。
2. `get_data()` 开始执行，遇到 `await asyncio.sleep(2)`，暂停，事件循环转去执行 `other_task()`。
3. `other_task()` 执行到 `await asyncio.sleep(1)`，暂停，此时两个任务都在等待，事件循环等待 1 秒。
4. 1 秒后，`other_task()` 恢复，打印“其他任务完成”并结束。
5. 再等 1 秒（总共 2 秒），`get_data()` 恢复，打印“数据请求完成”并结束。

**总耗时约 2 秒**（而同步执行需要 3 秒），效率明显提升。

## 案例

通过异步编程方式，实现文件上传的小功能。大致思路，从终端获取指定文件路径（一个或多个），然后通过异步的方式同时读取多个文件，并保存在当前文件夹中。

### 使用生成器读取文件内容

使用生成器读取文件内容，每次调用读取一行，避免大文件导致的内存溢出问题。

```python
def file_reader_generator(file_path):
    try:
        with open(file_path, "r", encoding="utf-8") as file:
            for line in file:
                yield line # 暂停运行，返回一行，等再次调用，返回下一行
    except Exception as e:
        print(f"读取文件 {file_path} 时出错: {e}")
```

### 将读取到的内容保存到文件中

采取流式读取，也就是一边读取文件内容，一边保存到指定文件中。

```python
async def save_content(original_file_path):
    original_path = Path(original_file_path)
    new_filepath = f"save_{original_path.name}"

    with open(new_filepath, "w", encoding="utf-8") as file:
        for line in file_reader_generator(original_file_path):
            file.write(line)

    print(f"文件已保存到: {new_filepath}")
```

### 以异步的方式处理文件

``python
async def process_files(file_paths):
    tasks = []
    for file_path in file_paths:
        if os.path.exists(file_path):
            tasks.append(
                asyncio.create_task(save_content(file_path))
            )
        else:
            print(f"文件不存在: {file_path}")
    
    await asyncio.gather(*tasks)
```

### 完整代码

``python
import asyncio
import sys
import os
from pathlib import Path

def file_reader_generator(file_path):
    """
    使用生成器读取文件内容，避免大文件导致的内存溢出问题
    """
    try:
        with open(file_path, "r", encoding="utf-8") as file:
            for line in file:
                yield line
    except Exception as e:
        print(f"读取文件 {file_path} 时出错: {e}")


async def save_content(original_file_path):
    """将内容保存到新文件中"""
    original_path = Path(original_file_path)
    new_filepath = f"save_{original_path.name}"

    with open(new_filepath, "w", encoding="utf-8") as file:
        for line in file_reader_generator(original_file_path):
            file.write(line)

    print(f"文件已保存到: {new_filepath}")


async def process_files(file_paths):
    """处理所有文件"""
    tasks = []
    for file_path in file_paths:
        if os.path.exists(file_path):
            tasks.append(
                asyncio.create_task(save_content(file_path))
            )
        else:
            print(f"文件不存在: {file_path}")
    
    await asyncio.gather(*tasks)


if __name__ == "__main__":
    # 从终端获取文件路径参数
    if len(sys.argv) < 2:
        print("请提供至少一个文件路径作为参数")
        print("用法: python main.py file1.txt file2.txt ...")
        sys.exit(1)

    file_paths = sys.argv[1:]
    asyncio.run(process_files(file_paths))
```

## 总结

- `yield` 用于生成器，实现迭代器协议，主要用于数据生成和懒加载
- `async` 和 `await` 用于异步编程，实现非阻塞 I/O 操作，提高程序并发性能
- `async/await` 通常与 `asyncio` 模块配合使用，构建异步应用

## 参考

- [异步IO | 廖雪峰的官方网站](https://liaoxuefeng.com/books/python/async-io/index.html)
- [Python asyncio 模块 | 菜鸟教程](https://www.runoob.com/python3/python-asyncio.html)