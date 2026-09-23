---
title: 一个下载器的工程化踩坑：单文件 .bat、cmd 陷阱与 uv 锁源
date: 2026-09-23
tags: [Python, uv, 工程化, Windows]
description: 功能写完只算一半。真正难的是把程序做成「双击即用」——从一个自解压的单文件 .bat，到 cmd 的重定向陷阱、Python 版本语法差异、uv.lock 的源绑定，以及 Windows 控制台的编码问题。
---

做一个小工具，功能写出来只算完成一半。另一半是**分发**：怎么让拿到它的人不用配环境、不用看文档，双击就能跑起来。

vidgrab 走的是「Windows 优先、零依赖启动」的路线，为了做到这一点踩了不少坑。这篇就聊聊这些工程细节，每一条都是真实撞出来的。

## 先把整份程序塞进单个 .bat

项目提供两种用法：文件夹版（`run.bat` + 源码）和**单文件版**（只有一个 `vidgrab-standalone.bat`）。后者只需要一个文件就能运行，它的原理是「批处理脚本 + 内嵌载荷」：

```text
1. 用 zipapp 把 src/vidgrab 打成单文件应用 vidgrab.pyz
2. 把 .pyz 做 base64 编码，内嵌到 .bat 末尾（纯 ASCII，不受编码影响）
3. .bat 运行时：
     findstr 定位载荷行 → more 跳过前面的批处理代码提取 base64
     → certutil -decode 还原 vidgrab.pyz
     → uv run --with yt-dlp ... python vidgrab.pyz
```

对应的批处理核心几行：

```bat
for /f "delims=:" %%i in ('findstr /n /b /e /c:"rem __VIDGRAB_PAYLOAD__" "%~f0"') do set "SKIP=%%i"
more +%SKIP% "%~f0" > "%B64%" 2>nul
certutil -f -decode "%B64%" "%PYZ%" >nul 2>nul
"%UV%" run --with "yt-dlp[default]" --with "curl_cffi" --with "imageio-ffmpeg" python "%PYZ%"
```

依赖不打包，而是运行时用 `uv run --with` 按需拉取——省掉了维护体积巨大的内嵌依赖，代价是首次要联网下载约 40MB。

### 但构建产物会过期：用 `--check` 卡住

单文件 `.bat` 是**构建产物**，一旦改了源码却忘了重新生成，发出去的就是旧版本。所以有个 `--check` 模式：从已提交的 `.bat` 里还原出 `.pyz`，逐文件比对是否与当前源码一致，**不一致就非零退出**，直接卡在 CI 里。

```text
x vidgrab-standalone.bat 与当前源码不一致，请重新生成：
    已过期 vidgrab/cli.py
```

## 踩坑 1：cmd 注释里的 `->` 会真的创建文件

批处理里写 `rem 双击 -> 打开菜单` 看起来人畜无害。但 `cmd` 会把 `rem`/`echo` 行里**没有被引号包住、也没用 `^` 转义**的 `<` `>` `|` `&` 当作重定向符——于是真的会在磁盘上创建一个名为「打开菜单」的空文件。

作者的做法是写了个审计函数，构建时扫描所有 `rem`/`echo` 行并对危险字符报错：

```python
_DANGER_RE = re.compile(r"(?<!\^)[<>|&]")

def audit_bat(text: str, label: str) -> list[str]:
    ...
    stripped = re.sub(r'"[^"]*"', '""', st)   # 引号内的字符是安全的
    for m in _DANGER_RE.finditer(stripped):
        problems.append(f"{label} 第 {i} 行出现 {m.group()!r}：{st[:70]}")
```

后来统一把 `->` 换成了全角 `→`，并把审计挂进构建自检——`cmd` 的坑也适用于 Windows，不只是老式 DOS。

## 踩坑 2：Python 3.9 不认 f-string 里的多行表达式

项目声明支持 Python **3.9+**，但下面这种写法只有 **3.12+（PEP 701）**才合法：

```python
# ❌ 3.9~3.11 报 "EOL while scanning string literal"
say(f"    {dim(_s("（输入 0 或 q 可退出）",
                  "(0 or q to quit)"))}")
```

3.9~3.11 的 f-string 里不能出现跨行的表达式，也不能复用同种引号。所以把 `_s(...)` 提到 f-string 外面：

```python
hint = _s("（输入 0 或 q 可退出）", "(0 or q to quit)")
say(f"    {dim(hint)}")
```

问题在于：`compileall` 这类**语法检查也发现不了**——它是运行时才炸。于是项目补了一个不联网、不需要 pytest 的冒烟测试，专门覆盖这类跨版本问题。

## 踩坑 3：uv.lock 和镜像源是绑定的

开发机的 `pypi.org` 被 DNS 解析到 fake-ip、索引页只有 ~15 KB/s，而阿里云镜像是 ~1.6 MB/s。所以项目默认走阿里云镜像。但代价是：

> **`uv.lock` 会记录每个包是从哪个索引解析来的**——当前锁文件里的引用全部指向阿里云镜像。

于是在用官方源的环境里执行 `uv sync`，uv 会判定锁文件过期、重新解析并改写 `uv.lock`（`git status` 变脏）。CI 的对策是显式对齐镜像源：

```yaml
env:
  UV_DEFAULT_INDEX: https://mirrors.aliyun.com/pypi/simple/
```

这样 CI 和本地看到的是同一个索引，锁文件才稳定。

## 踩坑 4：Windows 控制台的 ANSI 和 UTF-8

这是两个独立的问题。

其一，传统 `cmd.exe` 默认不解析 ANSI 转义序列，不开启的话彩色输出会变成 `←[32m` 这类乱码，进度条也无法原地刷新。需要手动开启 VT 处理：

```python
# ENABLE_VIRTUAL_TERMINAL_PROCESSING = 0x0004
kernel32.SetConsoleMode(handle, mode.value | 0x0004)
```

其二，默认 GBK 代码页下中文会乱码，需要强制 UTF-8：

```python
stream.reconfigure(encoding="utf-8", errors="replace")
```

此外还留了一个纯 ASCII 模式（`VIDGRAB_ASCII=1`），给字体缺字或乱码的老终端兜底。

## 让「双击」真的零门槛

一个「双击即用」的工具，难点其实在**环境准备**。`run.bat` 做了三级兜底，尽可能不让用户手动干活：

```text
装 uv：winget  →  pip  →  官方安装脚本
同步依赖：阿里云镜像  →  清华镜像  →  退回最小依赖（不含内置 ffmpeg）
```

CI 也用矩阵在三套环境上验证，尤其是声明的最低版本和主目标平台：

```yaml
matrix:
  include:
    - { os: ubuntu-latest,  python: "3.9"  }   # 声明的最低版本
    - { os: ubuntu-latest,  python: "3.13" }
    - { os: windows-latest, python: "3.13" }   # 主要目标平台
```

跑的不只是编译：语法检查、导入检查、冒烟测试、CLI 冒烟，以及「单文件版是否与源码同步」。那个不联网、不依赖 pytest 的冒烟测试，正好补上 `compileall` 覆盖不到的运行时问题（比如上面那个 3.9 的 f-string 坑）。

## 小结

这个项目的功能不复杂，本质是给 `yt-dlp` 套壳。但要真正做到「双击即用」，得同时处理好这几类问题：

- **分发**：单文件自解压 + `--check` 防止产物过期；
- **平台怪癖**：`cmd` 的重定向陷阱、Windows 控制台编码与 ANSI；
- **跨版本兼容**：低版本 Python 的语法差异，且静态检查发现不了，只能靠运行时测试；
- **依赖可复现**：`uv.lock` 与索引源绑定，需要本地和 CI 对齐。

比起「写个脚本调 yt-dlp」，这些才是把它变成**给别人也能用的工具**所需要的部分。

项目地址：[liku-yu/video-downloader](https://github.com/liku-yu/video-downloader)（MIT License）。

**相关阅读：**
- [把 yt-dlp 封装成好用的下载器：vidgrab 的设计笔记](/posts/vidgrab-design/) —— 分层设计与配置驱动的另一半
- [用 Minecraft 红石入门数字电路](/posts/redstone-digital-logic/)
