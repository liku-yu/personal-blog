---
title: yt-dlp 技术解析：一条视频是怎么被解析、选择并下载的
date: 2026-09-20
tags: [yt-dlp, Python, 命令行]
description: yt-dlp 不只是一个下载命令，而是一条「站点解析 → 格式选择 → 分片下载 → 后处理」的管线。这篇拆开它的内部结构，重点讲清格式选择这门迷你语言，以及 ffmpeg 到底负责哪一步。
---

很多人对 `yt-dlp` 的印象是「一条命令下载视频」。但它其实是一条完整的管线：从 URL 匹配到对应的站点解析器，抽取出结构化的元数据，再从几十种视频流里按规则挑出要下的那几个，最后交给下载器和 ffmpeg 完成合并。

理解这条管线，比背参数有用得多。这篇就把它拆开看。

{{< callout title="免责声明" type="warning" >}}
本文涉及的工具仅用于下载**你有权保存的内容**。请遵守目标平台的服务协议与当地版权法律，不要用它绕过 DRM 或付费墙。
{{< /callout >}}

## 一条视频的旅程

```text
URL
 │
 ▼
① Extractor（站点解析器）── 匹配站点，抽取统一的 info_dict
 │
 ▼
② formats 列表 ── 一个视频往往有几十种流（分辨率/编码/容器/是否含音轨）
 │
 ▼
③ 格式选择（-f / -S）── 从 formats 里挑出要下载的流
 │
 ▼
④ Downloader ── 直链下载，或按分片（HLS/DASH）并发下载
 │
 ▼
⑤ PostProcessor ── 用 ffmpeg 合并音视频、转码、嵌入字幕/封面/元数据
 │
 ▼
输出文件
```

下面逐段看。

## ① Extractor：把每个网站翻译成统一的 info_dict

yt-dlp 内置上千个 **extractor（站点解析器）**，每个负责一个（或一类）网站。它的职责是把网页或接口返回的内容，翻译成一份**统一结构的 `info_dict`**——这是整条管线的通用货币。

因为平台差异被 extractor 吸收掉了，后面的格式选择、下载、后处理才能对所有站点一视同仁。

```bash
yt-dlp --list-extractors        # 列出所有解析器
yt-dlp -F "<链接>"               # 列出该视频的全部 formats
yt-dlp -J "<链接>"               # 打印完整 info_dict（JSON）
```

`info_dict` 里比较关键的字段：

| 字段 | 含义 |
| --- | --- |
| `id` / `title` | 视频 id 与标题 |
| `uploader` / `channel` | 作者 / 频道 |
| `duration` / `upload_date` | 时长 / 发布日期 |
| `formats` | **可用格式列表**（后文的核心） |
| `entries` | 播放列表的条目列表 |
| `webpage_url` / `url` | 页面地址 / 媒体直链 |
| `extractor_key` | 命中它的解析器名 |

播放列表会体现在 `entries` 上。默认会对列表里每个视频做完整解析，如果只想快速拿到条目列表（不解析每个视频），可以用：

```bash
yt-dlp --flat-playlist -J "<播放列表链接>"
```

代价是部分元数据会缺失——这也是为什么工具里常见「先 flat 拿列表、再按需补全」的两段式做法。

## ② formats：一个视频为什么有几十种流

以 YouTube 为例，同一支视频通常提供多种**分辨率 × 编码 × 容器**的组合，而且**视频流和音频流往往是分开的**（DASH 协议的特性）：

```text
137  mp4  1920x1080  avc1     (仅视频)
248  webm 1920x1080  vp9      (仅视频)
140  m4a  audio      aac      (仅音频)
251  webm audio      opus     (仅音频)
18   mp4  640x360    avc1+aac (视频+音频合体)
```

- **format code**（如 `137`）是 extractor 特定的，不同站点不通用；
- 高清画质几乎都是「视频流 + 音频流」分离的，**想要一个能播放的文件就必须合并**——这正是 ffmpeg 存在的意义；
- 低清往往有「音视频合体」的格式，不依赖 ffmpeg 也能直接下。

## ③ 格式选择：一门迷你语言

### 默认行为

不传任何参数时，yt-dlp 的默认选择等价于：

```bash
yt-dlp -f "bestvideo*+bestaudio/best"
```

含义是：优先选「最佳含视频的流」+「最佳纯音频流」并合并；如果前者已经自带音频，就直接用它；实在不行退回最佳合体格式。

> 注意 `bv*` 和 `bv` 的区别：`bv*`（带星号）表示「含视频的流，可能也有音频」，`bv` 则要求**纯视频**。

### 特殊名称

| 选择器 | 含义 |
| --- | --- |
| `best` / `b` | 同时含视频和音频的最佳格式 |
| `bestvideo` / `bv` | 最佳**纯视频**格式 |
| `bestvideo*` / `bv*` | 最佳**含视频**格式（可能也含音频） |
| `bestaudio` / `ba` | 最佳**纯音频**格式 |
| `worst` / `w` | 同时含视频和音频的最差格式 |
| `all` | 所有格式（配合过滤器用） |
| `mp4` / `webm` | 该扩展名的最佳单文件格式 |
| `137` | 指定 format code（站点相关） |

### 三个运算符

| 运算符 | 作用 | 例子 |
| --- | --- | --- |
| `+` | **合并**多个流（需 ffmpeg） | `bv+ba` |
| `/` | **回退**：左边优先，不可用则用右边 | `22/17/18` |
| `,` | **多选**：全部下载（而非只挑一个） | `137,140` |

组合起来可以表达相当复杂的偏好，例如「优先 1080p 以内、mp4 封装」：

```bash
yt-dlp -f "bv*[height<=1080][ext=mp4]+ba[ext=m4a]/b[height<=1080]"
```

### 过滤器：用 `[...]` 写条件

方括号里可以放条件。数值字段支持 `< <= > >= = !=`：

```bash
yt-dlp -f "bv[height<=?720][tbr>500]"
```

常用的数值字段有 `height`、`width`、`fps`、`filesize`、`tbr`、`abr`、`vbr`、`asr`、`audio_channels` 等。

字符串字段则支持 `=`（等于）、`^=`（前缀）、`$=`（后缀）、`*=`（包含）、`~=`（正则），并可用 `!` 取反：

```bash
yt-dlp -f "bv[vcodec^=avc1]"      # H.264 编码
yt-dlp -f "ba[ext=m4a]"           # m4a 音频
```

一个容易被忽略的细节：**字段值未知时，该格式会被排除**。如果希望「未知也算通过」，在比较符后加 `?`：

```bash
yt-dlp -f "bv[height<=?720]"      # 未知高度的也纳入候选
```

### `-S`：换一种更清晰的思路

用 `-f` 写「最差画质」是个常见误区：`worst` 选的是**所有维度都最差**的格式，而不是体积最小的。更合理的做法是用 `-S`（`--format-sort`）**定义什么叫「最好」**：

```bash
yt-dlp -S "+size,+br"        # 体积最小
yt-dlp -S "res:720"          # 不超过 720p 的最佳（没有更小的就选最小的）
yt-dlp -S "res:1080,vcodec:avc1,fps"   # 多字段依次排序
```

排序规则：

- 字段默认**降序**，前缀 `+` 改为升序（`+size` = 越小越优先）；
- `field:value` 表示「偏好接近该值」（`res:720` 即不超过 720p 的更大者）；
- `field~value` 则偏好「最接近」该值（`filesize~1G`）。

默认排序字段是 `lang,quality,res,fps,hdr:12,vcodec,channels,acodec,size,br,asr,proto,ext,hasaud,source,id`，其中 `hdr:12` 意味着默认不优先 Dolby Vision（兼容性考虑）。

{{< callout title="Tip" type="tip" >}}
想看清 yt-dlp 到底怎么排的，用 `yt-dlp -v -F "<链接>"`——它会按「最差 → 最好」把 formats 打印出来，正好可以验证你的 `-S` 有没有按预期生效。
{{< /callout >}}

## ④ Downloader：分片、并发与反爬

拿到选中的流后，下载器开始工作。要点：

- **分片下载**：HLS/DASH 视频由许多小分片组成。`-N/--concurrent-fragments` 可以并发拉分片，**默认是 1**，适当调大能显著提速：

```bash
yt-dlp -N 8 "<链接>"
```

- **断点续传**：默认会写 `.part` 临时文件，中断后重跑可续传（`--no-part` 可关闭）。
- **限速与礼貌请求**：面对有风控的站点，反爬手段通常是组合拳：

```bash
yt-dlp --limit-rate 2M --sleep-interval 3 --retries 20 \
       --cookies-from-browser chrome --impersonate chrome "<链接>"
```

其中 `--cookies-from-browser` 直接读取浏览器登录态（B 站高清、需要登录的内容都靠它），`--impersonate` 则伪装浏览器 TLS 指纹。

## ⑤ PostProcessor：ffmpeg 真正负责的部分

下载只是拿到一堆流，**后处理**才把它们变成一个正常文件。常见的后处理器：

| 选项 | 作用 |
| --- | --- |
| `--merge-output-format mp4` | 合并音视频时使用指定容器 |
| `-x` / `--extract-audio` | 只保留音频 |
| `--audio-format mp3` | 音频转码为 mp3（依赖 ffmpeg） |
| `--embed-metadata` | 写入标题/作者等元数据 |
| `--embed-thumbnail` | 把封面嵌入文件 |
| `--embed-subs` | 把字幕嵌入视频 |
| `--remux-video` / `--recode-video` | 重封装 / 重编码 |
| `--sponsorblock-remove` | 按 SponsorBlock 数据剪掉赞助片段 |

这些操作**几乎都由 ffmpeg 完成**。所以「ffmpeg 没装」的后果不是不能下载，而是高清视频**无法合并**、无法转码——你会得到分离的视频流和音频流。用 `--ffmpeg-location` 可以指定 ffmpeg 路径。

## CLI 只是 Python API 的一层壳

命令行所有选项，本质上都是在往一个 `params` 字典里写值，然后交给 `YoutubeDL`：

```python
from yt_dlp import YoutubeDL

opts = {
    "format": "bv*+ba/b",
    "outtmpl": "%(title).120s [%(id)s].%(ext)s",
    "progress_hooks": [lambda d: print(d.get("status"))],
}

with YoutubeDL(opts) as ydl:
    info = ydl.extract_info(url, download=False)  # 只解析，不下载
    ydl.download([url])                            # 执行下载
```

几个关键的扩展点：

- **`progress_hooks`**：下载进度回调，拿到 `downloaded_bytes` / `total_bytes` / `speed` / `eta`；
- **`postprocessor_hooks`**：后处理阶段回调，能拿到合并后的 `filepath`；
- **`logger`**：接管日志输出。**特别注意**：设置了 `ignoreerrors=True` 后，下载失败不会抛异常，错误只会走 logger 的 `error()`——不接管 logger 就会把失败误判成成功；
- **`postprocessors`**：用字典声明后处理链，效果等同于上面的 CLI 选项。

这也是各类「给 yt-dlp 套壳」的工具（包括我自己的 [vidgrab](/posts/vidgrab-design/)）普遍采用的接入方式。

## 实用速查

| 需求 | 命令 |
| --- | --- |
| 只看信息不下载 | `yt-dlp -J <url>` |
| 列出可用格式 | `yt-dlp -F <url>` |
| 限高 1080p | `yt-dlp -S "res:1080" <url>` |
| 只要音频 | `yt-dlp -x --audio-format mp3 <url>` |
| 批量 + 跳过已下 | `yt-dlp -a urls.txt --download-archive done.txt` |
| 登录态 | `yt-dlp --cookies-from-browser chrome <url>` |
| 分片加速 | `yt-dlp -N 8 <url>` |

## 小结

yt-dlp 的强大，来自它把「解析 → 格式选择 → 下载 → 后处理」清晰分层：

- **Extractor** 吸收各站点差异，产出统一的 `info_dict`；
- **格式选择**是一门小而完整的表达式语言：`+`/`/`/`,` 三个运算符、`[...]` 过滤器、以及更清晰的 `-S` 排序；
- **Downloader** 处理分片、并发与反爬；
- **PostProcessor** 借助 ffmpeg 把流变成能播放的文件。

理解这套结构之后，那些看起来密集的参数就有了共同的心智模型——你不必记住每一条，只需要知道「我现在在哪一层」。

**相关阅读：**
- [把 yt-dlp 封装成好用的下载器：vidgrab 的设计笔记](/posts/vidgrab-design/)
- [一个下载器的工程化踩坑：单文件 .bat、cmd 陷阱与 uv 锁源](/posts/vidgrab-engineering/)
