---
title: Agent 的工程细节：上下文裁剪、重试、编码与打包
date: 2026-09-30
tags: [Python, 工程化, Windows]
description: 核心循环之外，真正决定一个 Agent 能不能用的，是这些琐碎却关键的问题：历史无限增长怎么办、流式中断该不该重试、Windows 控制台为什么乱码、TUI 怎么和核心解耦，以及怎么打包成一个 exe。
---

[第一篇](/posts/agent-from-scratch/)把 Agent 的核心循环跑通了，[第二篇](/posts/agent-command-safety/)加了安全护栏。但一个「能给自己用」的脚本，和一个「能打包给别人用」的工具，中间隔着的正是这篇要讲的东西。

它们都不涉及模型能力，却决定这个 Agent 能不能稳定地用下去。

## 上下文预算：历史不能无限增长

Responses API 是无状态的，每一轮都要把**完整历史**发回去。而 Agent 的工具调用会产生大量输出（命令结果、文件内容……），几轮之后请求体就会膨胀到一个危险的大小。

于是需要一个粗糙但有效的预算机制：用序列化后的字符数估算体积，超过阈值就裁剪：

```python
MAX_CONTEXT_CHARS = 200_000

def _items_size(items):
    return sum(len(json.dumps(it, ensure_ascii=False, default=str)) for it in items)
```

**裁剪的难点在于不能随便切。** 历史不是一串独立消息，而是有依赖关系的 item 序列——`function_call` 和它的 `function_call_output` 必须成对存在，`call_id` 要能对上。如果从中间截断，就会留下一个「有调用、没结果」的悬空结构，模型可能因此报错或行为异常。

所以裁剪的单位是**「整轮对话」**——按用户消息切分，最旧的整轮优先丢弃：

```python
def _compact(input_items):
    if len(input_items) <= 2 or _items_size(input_items) <= MAX_CONTEXT_CHARS:
        return
    system = input_items[0]
    groups, cur = [], []
    for it in input_items[1:]:
        if it.get("type") == "message" and it.get("role") == "user":
            if cur:
                groups.append(cur)      # 遇到新的用户消息 = 上一轮结束
            cur = [it]
        else:
            cur.append(it)              # 工具调用/结果归入当前轮
    if cur:
        groups.append(cur)

    while len(groups) > 1 and \
            _items_size([system] + [x for g in groups for x in g]) > MAX_CONTEXT_CHARS:
        groups.pop(0)                   # 丢掉最旧的一整轮

    input_items[:] = [system] + [x for g in groups for x in g]
```

system 永远保留，最少保留一轮，保证「成对」关系不被破坏。

## 重试：只在还没输出之前

网络抖动、429、5xx 都值得重试。先定义「什么算瞬时错误」：

```python
_TRANSIENT_STATUS = {408, 409, 429, 500, 502, 503, 504}

def _is_transient(exc):
    if isinstance(exc, httpx.TransportError):
        return True
    if isinstance(exc, httpx.HTTPStatusError):
        return exc.response.status_code in _TRANSIENT_STATUS
    return False
```

但流式请求的重试有个陷阱：**如果已经开始往终端输出，再重试就会把内容重复打印一遍**。所以重试必须加一个大前提——「本轮尚未产生任何事件」：

```python
started = False   # 一旦收到任何事件就置 True

# ...
if _is_transient(e) and not started and attempt < MAX_RETRIES:
    attempt += 1
    delay = min(2 ** attempt, 8)          # 指数退避，封顶 8 秒
    time.sleep(delay)
    continue
raise
```

这个 `started` 标志是个很典型的「流式接口才有的约束」——非流式请求随便重试都无所谓，流式就要为「已经吐出去的内容」负责。

## Windows 编码：两个独立的坑

这是整个项目里最 Windows 特有的部分，而且**中文环境下几乎必踩**。

**坑一：控制台代码页是 GBK。** 模型返回 UTF-8 文本，默认输出流按 GBK 编码，轻则乱码、重则遇到非 GBK 字符直接抛异常。解决办法是强制把 stdout/stderr 设为 UTF-8：

```python
for stream in (sys.stdout, sys.stderr):
    reconfigure = getattr(stream, "reconfigure", None)
    if reconfigure is not None:
        try:
            reconfigure(encoding="utf-8", errors="replace")
        except Exception:
            pass
```

**坑二：管道输入的代理字符。** 当 stdin 是管道时（如 `echo "..." | agent.py --one`），如果按 GBK 解码 UTF-8 字节，可能产生 **surrogate code point**（代理项）。这种字符串在 `json.dumps` 时会让整个请求构造失败——报错信息还很难懂。

处理方式是：**stdin 只在它是管道时才重设为 UTF-8**（交互式终端不碰），读取时也直接走 bytes 解码：

```python
def read_stdin_text() -> str:
    try:
        return sys.stdin.buffer.read().decode("utf-8", errors="replace").strip()
    except Exception:
        return sys.stdin.read().strip()
```

> 规律很清楚：**交互式输入交给终端，管道输入自己按 UTF-8 解码**。把这两者混在一起处理，就会踩到代理字符的坑。

## TUI 与核心解耦：一个 emit 回调

如果核心逻辑里直接写 `rich.print`，那它就再也离不开界面库了。这个项目的做法是：**核心只负责产出事件，渲染交给外层**。

核心定义了一个回调签名：

```python
Emit = Callable[[str, dict], None]
```

然后所有「想显示点什么」的地方都调用 `emit(事件名, 数据)`，而不是直接打印：

| 事件 | 含义 |
| --- | --- |
| `reasoning` / `reasoning_done` | 推理过程的增量与结束 |
| `text` / `text_done` | 最终回答的增量与结束 |
| `tool` | 一次工具执行（命令 + 结果 + 状态） |
| `notice` / `error` / `aborted` | 提示、错误、中断 |

`tui.py` 传入自己的 `emit` 实现，用 prompt_toolkit + Rich 渲染成带边框的版面；而 `--plain` 模式传 `None`，核心就退回直接写 stdout/stderr。**同一套逻辑，两种前端。**

TUI 还做了一层 `try/except ImportError` 的懒加载——没装 rich/prompt_toolkit 时自动退回纯文本 REPL，不会因为缺一个可选依赖就整个跑不起来。

另外有个 Git Bash 特有的守卫：终端如果缺少原生 Windows 控制台，prompt_toolkit 会崩溃。所以启动前先探测一下：

```python
if os.name == "nt" and "xterm" in os.environ.get("TERM", "") and not _has_windows_console():
    console.print(Text("TUI 需要原生 Windows 控制台。\n"
                       "  Git Bash 请用：winpty uv run python agent.py", style=ERROR))
    sys.exit(1)
```

`_has_windows_console()` 通过 `GetConsoleMode` 判断句柄是否是真控制台——因为 mintty 暴露的只是一个 xterm pty。

## 打包：单文件 exe，但 key 不入包

最后是分发。用 PyInstaller 打成一个 exe，而 TUI 依赖（rich、prompt_toolkit）需要显式收集，`tui` 还得作为隐藏导入，否则会被漏掉：

```python
from PyInstaller.utils.hooks import collect_all

hiddenimports = ['tui']
for pkg in ('rich', 'prompt_toolkit'):
    d, b, h = collect_all(pkg)
    datas += d; binaries += b; hiddenimports += h

a = Analysis(['agent.py'], hiddenimports=hiddenimports,
             excludes=['tkinter', 'unittest', 'pydoc'],
             optimize=2)
```

有一个安全红线：**API Key 绝不能打进二进制**。所以运行时才去读：

```python
def _key():
    env = os.environ.get("DEEPSEEK_API_KEY", "").strip()
    if env:
        return env
    # 打包成 onefile 后 __file__ 指向临时解压目录，
    # 所以还要在真正的 exe 旁边找 .env
    base = (os.path.dirname(os.path.abspath(sys.executable))
            if getattr(sys, "frozen", False)
            else os.path.dirname(os.path.abspath(__file__)))
    ...
```

`getattr(sys, "frozen", False)` 用来区分「源码运行」和「打包后运行」——onefile 模式下 `__file__` 指向临时目录，只有 `sys.executable` 才指向 exe 本身，`.env` 要放在它旁边才找得到。

构建脚本则封装成一条命令：

```bash
uv run pyinstaller --noconfirm --clean --distpath "$OUT" deepseek-agent.spec
```

## 小结

这些细节单独看都不起眼，但每一个都能让「能跑的 demo」变成「不想用的工具」：

- **上下文裁剪**要按整轮处理，因为工具调用是成对的结构；
- **流式重试**必须判断「是否已输出」，否则会重复渲染；
- **Windows 编码**要把交互输入和管道输入分开对待；
- **TUI 解耦**靠一个 `emit` 回调，核心不依赖渲染库；
- **打包**时显式收集可选依赖，并让密钥留在运行时读取。

写完这些，我才真正体会到：**Agent 的「智能」来自模型，但「可用」来自工程。**

项目地址：[liku-yu/deepseek-agent](https://github.com/liku-yu/deepseek-agent)。

**相关阅读：**
- [从零开始写一个 Agent：一次极简的尝试](/posts/agent-from-scratch/)
- [让 AI Agent 安全地执行命令](/posts/agent-command-safety/)
