---
title: 把 yt-dlp 封装成好用的下载器：vidgrab 的设计笔记
date: 2026-09-23
tags: [Python, yt-dlp, CLI]
description: yt-dlp 能力很强，但接口是给命令行高手用的。vidgrab 用一层薄封装把它变成「双击即用」的工具——这篇讲它的分层设计、配置驱动的选项构造、双层进度条、风控重试，以及一个容易踩的错误处理陷阱。
---

`yt-dlp` 大概是目前最强的开源视频下载器，内置上千个站点解析器。但它的接口是给命令行高级用户设计的：合并音视频要 ffmpeg、B 站高清要登录态、分片要并发数、下不动要换代理、批量要写文件……对不常用命令的人来说，门槛不低。

**vidgrab** 就是给它套的一层壳：基于 yt-dlp 内核，用 `uv` 管理依赖，双击一个 `.bat` 就能用。这篇不讲怎么用，而是讲讲这层壳在设计上做对了哪些事。

{{< callout title="免责声明" type="warning" >}}
本工具仅用于下载**你有权保存的内容**。请遵守目标平台的服务协议与当地版权法律。项目不包含、也不支持任何绕过 DRM 或付费墙的代码。
{{< /callout >}}

## 它长什么样

交互式菜单是主要入口，任何提问处输入 `0` 或 `q` 都能安全退出、不下载任何东西：

```text
  ▸ 1. 下载链接                2. 编辑 urls.txt 链接清单并批量下载
    3. 仅下载音频（MP3）        4. 查看链接信息与可用画质
    5. 按关键词搜索并下载        6. 快速设置
    7. 查看支持的站点           8. 打开下载目录         9. 退出程序
```

同时保留完整的命令行入口：

```bat
uv run python main.py "<链接>"                                   :: 最高画质
uv run python main.py -q 1080p -o "D:/Videos" "<链接1>" "<链接2>"
uv run python main.py --audio --audio-format mp3 "<链接>"
uv run python main.py -f urls.txt --jobs 3                       :: 批量 + 3 并发
uv run python main.py --search "关键词" --search-site bilibili --search-limit 5
```

## 架构：四层 + 一个贯穿全局的 Settings

```text
src/vidgrab/
├─ config.py    # 纯数据：Settings dataclass + 画质/音频/搜索预设表
├─ core.py      # 下载核心：把 Settings 翻译成 yt-dlp 选项、批量调度、搜索
├─ cli.py       # argparse 命令行 + 交互菜单
└─ ui.py        # 终端适配：ANSI/ASCII、进度条、菜单组件
```

关键设计是：**所有可调参数都收进一个 `Settings` dataclass**。

命令行解析出的参数、菜单里选择的项，最终都只是往 `Settings` 里写值；`core.Downloader` 只认 `Settings`，不关心它从哪来。好处很直接——菜单和命令行不会出现「命令行能做、菜单做不到」的割裂；加一个新参数，只需要在 `Settings` 里加一个字段、在构造 yt-dlp 选项时读一次。

## 设计一：配置驱动，把 Settings 翻译成 yt-dlp options

yt-dlp 的 Python API 选项很多，硬编码在各处会迅速失控。vidgrab 把它们集中到一个 `build_opts()` 里构造，画质档位则用一张预设表维护：

```python
QUALITY_FORMATS = {
    "best":  "bv*+ba/b",   # 最高画质（自动合并音视频）
    "1080p": "bv*[height<=1080]+ba/b[height<=1080]",
    "720p":  "bv*[height<=720]+ba/b[height<=720]",
    "worst": "wv*+wa/w",   # 最小体积
}
```

| 档位 | format 表达式 |
| --- | --- |
| `best` | `bv*+ba/b` |
| `1080p` | `bv*[height<=1080]+ba/b[height<=1080]` |
| `720p` | `bv*[height<=720]+ba/b[height<=720]` |
| `worst` | `wv*+wa/w` |

后处理器链同样是声明式地拼出来的：

```python
pps = []                       # postprocessors
if s.audio_only:
    pps.append({"key": "FFmpegExtractAudio",
                "preferredcodec": s.audio_format,
                "preferredquality": s.audio_quality})
if s.embed_metadata:
    pps.append({"key": "FFmpegMetadata"})
if s.embed_thumbnail:
    pps.append({"key": "EmbedThumbnail"})
if s.embed_subs:
    pps.append({"key": "FFmpegEmbedSubtitle"})
opts["postprocessors"] = pps
```

**好处是各功能开关彼此独立**，组合起来就是完整能力，不需要为每种场景写一条分支。

## 设计二：进度条要算「整批总进度」

yt-dlp 的 `progress_hooks` 只给**当前文件**的进度。但批量下载时，用户真正关心的是「这一批下到哪了」。所以进度条做了两层：

```text
[1/2] [████████████░░░░░░░░░░░░]  50.4%  2.0MiB/4.0MiB  3.1MiB/s  ETA 00:00  │ 总  25.2%
```

`50.4%` 是当前文件，`│ 总 25.2%` 是按已完成任务数加权的整批进度。两个细节值得说：

1. **总进度只增不减**：同一个视频会分「视频流 / 音频流」两次下载，进度会重置。不处理的话总进度会来回跳。代码里记录历史峰值，下降时保持峰值不动。
2. **按输出目标切换渲染策略**：终端（TTY）用 `\r` 原地刷新；一旦重定向到文件或管道，就按 **10% 分档**节流输出，避免刷屏。

## 设计三：搜索要「flat 优先 + 带退避的重试」

关键词搜索形如 `bilisearch10:关键词`，实现上有两个关键决策。

**其一，flat 优先**。先用 `extract_flat` 一次请求拿到候选列表，再对缺标题的条目小并发补全。因为某些平台的 flat 搜索只返回 id（B 站就是一串 av 号），而直接非 flat 搜索又会因连续快速请求被风控——先轻量拿列表、再按需补全，是折中。

**其二，把风控当成概率事件**。B 站搜索接口会间歇性返回 HTTP 412，实测相同参数连续请求成功率约 4/6。所以失败后不是直接报错，而是等待并重试：

```python
for attempt in range(1, attempts + 1):
    try:
        ...
        break
    except Exception as exc:
        last_exc = exc
        if attempt < attempts:
            time.sleep(1.5 * attempt)   # 1.5s / 3s 退避
```

## 设计四：ignoreerrors 的陷阱

这是个很容易写错的地方。yt-dlp 的 Python API 里，一旦设置 `ignoreerrors=True`，**下载失败不会抛异常**——错误只会从你传入的 logger 的 `error()` 方法里冒出来。如果只检查 `ydl.download()` 的返回值，失败任务会被误判为成功。

做法是自定义 logger，把错误收集进一个 `sink`：

```python
class _Logger:
    def error(self, msg: str) -> None:
        # ignoreerrors=True 时不抛异常，错误只从这里出来，
        # 必须记录，否则失败任务会被误判为成功
        self.sink.setdefault("errors", []).append(msg)
```

更进一步，它区分了**整体失败**和**部分成功**：播放列表里有些条目成功、有些失败时，`TaskResult` 标记为 `partial`，汇总时单独统计，而不是笼统算作失败。

## 小结

vidgrab 的功能本质上是「给 yt-dlp 套壳」，但几个设计决策让它明显比「写个脚本调 yt-dlp」更成熟：

- **一个 `Settings` 贯穿全局**：命令行和菜单共用一套逻辑，扩展成本低；
- **配置驱动**：画质、音频、字幕、元数据都是独立开关，组合自由；
- **贴着实况做交互**：总进度只增不减、风控退避重试、部分成功单独统计；
- **不放过 API 的坑**：`ignoreerrors` 下错误只从 logger 来，漏了就会误判。

项目地址：[liku-yu/video-downloader](https://github.com/liku-yu/video-downloader)（MIT License）。

**相关阅读：**
- [一个下载器的工程化踩坑：单文件 .bat、cmd 陷阱与 uv 锁源](/posts/vidgrab-engineering/) —— 分发与跨平台的另一半故事
- [用 Markdown 写文章：Void 主题写作指南](/posts/mdx-writing-guide/)
