---
title: 从零开始写一个 Agent：一次极简的尝试
date: 2026-09-30
tags: [AI Agent, LLM, Python]
description: 不借助任何 agent 框架，只用一个 HTTP 客户端和一个工具，把「会调用工具的模型」从头实现一遍。这篇记录整个过程——Agent 到底比普通对话多了什么，那个循环长什么样。
---

市面上的 Agent 框架越来越多，封装的层次也越来越厚。看多了反而有个疑问挥之不去：**一个 Agent 剥离掉所有框架之后，剩下的核心到底是什么？**

为了搞清楚这件事，我用最朴素的方式从零写了一个：不引入任何 agent 框架，只有一个 HTTP 客户端、一个工具，和一个循环。项目叫 **deepseek-agent**——极简命令行 Agent，唯一能做的事就是**在 Git Bash 里执行命令**。

这篇记录整个尝试的思路。技术栈简单到有点无聊：`httpx` 负责请求，`bash` 是唯一的工具，剩下的全靠自己写。

{{< callout title="说明" type="neutral" >}}
本文讲的都是代码结构与设计，**不需要真实调用 API**。文中出现的 Key 一律脱敏为 `sk-****`。
{{< /callout >}}

## Agent 和聊天机器人的区别

先想清楚这件事，后面才有的写。

普通对话（chatbot）是**一问一答**：你发消息，模型回消息，结束。模型的知识和手都停在上一次训练里。

而 Agent 多了一件事：**模型可以调用工具，观察真实结果，然后决定下一步**。当你让它「看看当前目录里有什么」，它不是凭记忆猜，而是：

```text
模型：我要执行 `ls`
本地：执行 → 得到真实输出
模型：（看到输出）好的，目录里有 a.txt、b.txt…
```

也就是说，Agent = **模型 + 工具 + 一个循环**。剥掉所有框架，核心就是这个反馈回路：

```text
loop {
    把对话历史发给模型
    如果模型要求调用工具：
        执行工具，把结果追加进历史
        继续循环
    否则：
        模型给出了最终回答，结束
}
```

想通这一点，实现就变得具体了。

## 输入不再是一串消息，而是一组 items

这个项目用的是 **Responses API**（`POST /v1/responses`）。它和常见的 Chat Completions 有一个关键区别：请求体的 `input` 不是简单的 `messages`，而是一组带类型的 **items**：

```python
def _system_item():
    return {"type": "message", "role": "system",
            "content": [{"type": "input_text", "text": SYSTEM}]}

def _user_item(text):
    return {"type": "message", "role": "user",
            "content": [{"type": "input_text", "text": text}]}
```

一开始觉得这比 `{"role": "user", "content": "..."}` 啰嗦，但很快发现它是**为工具调用准备的**——因为工具调用和它的返回值也是 item：

```python
{"type": "function_call",        "call_id": "...", "name": "bash", "arguments": "{...}"}
{"type": "function_call_output", "call_id": "...", "output": "命令输出"}
```

整个 Agent 的历史就是这些 item 的列表。**`call_id` 把「模型发起的调用」和「我执行后的结果」配成一对**，这是后面回填的基础。

## 定义唯一的工具：bash

工具用 JSON Schema 描述，模型靠它知道「有这么个工具、参数长什么样」：

```python
TOOLS = [{
    "type": "function",
    "name": "bash",
    "description": "Run a shell command in Git Bash and return its stdout, stderr and exit code.",
    "parameters": {
        "type": "object",
        "properties": {
            "command": {"type": "string", "description": "The shell command to run."}
        },
        "required": ["command"],
    },
}]
```

只给一个 `bash` 是刻意的克制：它能做几乎所有事（读文件、跑测试、装依赖……），省掉了一堆专用工具的复杂度。当然，这也直接引出安全问题——留到[下一篇](/posts/agent-command-safety/)讲。

## 核心循环：二十几行就是全部

有了 items 和工具，主循环短得出乎意料：

```python
def run_agent(input_items, emit=None, confirm=None):
    while True:
        output_items = stream_call(input_items, TOOLS, emit)
        input_items.extend(output_items)              # 回填模型这一轮的输出

        tool_calls = [it for it in output_items if it.get("type") == "function_call"]
        if not tool_calls:
            return                                     # 没有工具调用 = 最终回答

        for call in tool_calls:                        # 执行每个工具调用
            result = execute_tool(call["name"], call["arguments"], emit, confirm)
            input_items.append({
                "type": "function_call_output",
                "call_id": call["call_id"],
                "output": result,
            })
```

几个设计选择：

- **不限轮数**：`while True` 一直转到模型不再调用工具。这是取舍——不设上限意味着一个复杂任务可以自主推进很多步；代价是万一模型「钻牛角尖」，得靠用户手动中断（见第二篇的取消机制）。
- **回填模型自己的输出**：`input_items.extend(output_items)` 不只是把工具结果塞回去，连模型这轮的 reasoning 和 function_call 也一并保留，下一轮它才能「记得」自己刚才做了什么。
- **每一轮都是完整历史**：Responses API 是无状态的，每次请求都要把全部 items 发过去。历史会越来越长，于是就有了[第三篇](/posts/agent-engineering-details/)要讲的上下文裁剪。

## 流式：让等待变得可感知

调用模型时如果不流式，用户就要盯着黑屏等好几秒。所以请求带 `stream: True`，响应是 SSE（Server-Sent Events），逐行读：

```python
with httpx.stream("POST", f"{API_BASE}/responses", headers=headers,
                  json=body, timeout=180) as resp:
    for line in resp.iter_lines():
        if not line or not line.startswith("data:"):
            continue
        data = line[5:].strip()
        if data == "[DONE]":
            continue
        evt = json.loads(data)                    # 解析事件
        t = evt.get("type")
```

事件类型里，最有意思的是它把**推理过程和最终回答分开**：

| 事件 | 含义 |
| --- | --- |
| `response.reasoning_text.delta` | 推理过程的增量（可以灰色流式展示） |
| `response.output_text.delta` | 最终回答的增量 |
| `response.output_item.done` | 一个完整 item 结束（收集起来回填用） |
| `response.completed` | 本轮结束，附带 usage |

这里有个**很容易踩的坑**：流式接口里，失败往往不是抛异常，而是发一个事件。如果不处理，程序会「安静地成功」——什么都没输出，却当成正常结束：

```python
# 必须显式把 API 侧失败变成异常
if t in ("response.failed", "error", "response.incomplete"):
    err = evt.get("error") or (evt.get("response") or {}).get("error") or t
    raise AgentError(f"{t}: {err}")
```

与此同理，网络抖动导致的瞬时错误要能重试，但**只在「还没输出任何内容」时**才重试，否则会把已经渲染的内容重复打印一遍——这个细节也放在第三篇展开。

## 工具执行：把命令跑起来

工具执行看着最简单，其实藏着不少东西。最朴素的版本是 `subprocess.run`，但它会在两个地方掉链子：**超时**和**取消**——只 kill 掉 bash 本身，它启动的子进程还会流浪。

所以这里手动管理进程，并让它跑在**独立的进程组**里，超时或取消时连同整棵进程树一起干掉：

```python
if os.name == "nt":
    popen_kwargs["creationflags"] = subprocess.CREATE_NEW_PROCESS_GROUP
else:
    popen_kwargs["start_new_session"] = True
```

返回值也做成了模型容易读的紧凑格式：

```text
exit_code=0
stdout:
...
stderr:
...
```

把 stdout/stderr 各截断到 4000 / 2000 字符，避免一次 `cat` 大文件就把上下文撑爆。

## 跑通之后，问题才刚开始

到这里，「模型 + 工具 + 循环」已经能用了：输入一句「用 bash 看看当前目录」，它就会真的执行 `ls` 并把结果讲给你听。核心代码加起来不到 200 行。

但一个能给别人用的 Agent，还有两座大山：

1. **安全**：你等于把 shell 交给了模型。危险命令、敏感文件、偷偷外传数据怎么办？靠一个正则黑名单显然不够，但完全没有防护更不行。
2. **工程**：历史无限增长、网络抖动重试、Windows 控制台中文乱码、TUI 与核心逻辑耦合、打包成单个 exe……这些和「模型聪不聪明」无关，却决定它能不能用。

这两块我都单独写了：

## 小结

回到最初的问题——**剥掉框架，Agent 的核心是什么？**

答案比想象中简单：**一个把「模型的工具调用请求」变成「真实执行结果」再喂回去的循环**。工具的定义是 JSON Schema，历史是一组带类型的 item，`call_id` 负责配对，流式负责体验。剩下的都是工程。

这个认识最大的价值是**去神秘化**：Agent 不是一个新物种，它就是一个反馈回路。理解了这个回路，再看各种框架，就只是「谁把这圈包得更舒服」的区别了。

项目地址：[liku-yu/deepseek-agent](https://github.com/liku-yu/deepseek-agent)。

**相关阅读：**
- [让 AI Agent 安全地执行命令](/posts/agent-command-safety/) —— 给模型 shell 权限时，如何加护栏
- [Agent 的工程细节：上下文裁剪、重试、编码与打包](/posts/agent-engineering-details/) —— 让这个循环真正可用的那些事
