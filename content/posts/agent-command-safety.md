---
title: 让 AI Agent 安全地执行命令
date: 2026-09-30
tags: [AI Agent, 安全, Python]
description: 给模型一个 bash 工具，等于把 shell 交到它手里。这篇讲一个最小 Agent 如何加护栏——危险命令三分类、审批与非交互拦截、环境变量剥离防外泄，以及 Ctrl+C 的取消语义和进程树清理。
---

在[上一篇](/posts/agent-from-scratch/)里，我给这个 Agent 只配了一个工具：`bash`。它因此能做几乎任何事，也意味着**我把 shell 交给了模型**。

这不是危言耸听。模型可能因为一次误判，执行 `rm -rf` 删掉不该删的目录；可能读走 `.env` 里的密钥；也可能在某些情况下把敏感信息通过 `curl` 发出去。给 Agent 执行能力的那一刻，安全就不再是可选项。

这篇讲这个最小 Agent 是怎么加护栏的。先说清楚：它是**启发式防护，不是沙箱**——但即便如此，也比毫无防护强得多。

## 三类需要警惕的命令

与其维护一张巨大的黑名单，不如先想清楚「什么命令值得停下来问一句」。这个项目归纳成三类：

| 类别 | 含义 | 例子 |
| --- | --- | --- |
| `destructive` | 可能造成不可逆破坏 | `rm -rf`、`mkfs`、`dd if=`、fork 炸弹 |
| `sensitive-file` | 触碰密钥/凭据文件 | `.env`、`.ssh/`、`id_rsa`、`.aws/credentials` |
| `network-egress` | 可能把数据传出去 | `curl`、`wget`、`ssh`、`nc` |

分类结果是一个列表，只要非空就进入审批流程：

```python
def classify_command(command: str) -> list[str]:
    if not command:
        return []
    s = _normalize(command)
    reasons = []
    if _rm_recursive_force(s) or _DESTRUCTIVE_RE.search(s):
        reasons.append("destructive")
    if _SENSITIVE_RE.search(command):
        reasons.append("sensitive-file")
    if _has_network_egress(s):
        reasons.append("network-egress")
    return reasons
```

## 匹配之前先「归一化」

命令匹配最容易被绕过的地方，是各种等价写法：`rm -rf x` 和 `rm -fr x` 是一回事，`'rm'  -RF   x` 加引号和多余空格也能跑。所以匹配前先做归一化——**去掉引号与反斜杠、压缩空白、转小写**：

```python
def _normalize(command: str) -> str:
    s = re.sub(r"[\"'`\\]", "", command)
    return re.sub(r"\s+", " ", s).lower()
```

`rm -rf` 还单独解析了一遍，因为它的参数顺序和分片写法很灵活。做法是**先按 `;&|` 把命令切成段**，只看每段开头的 `rm`，再检查是否同时带上递归和强制标志：

```python
def _rm_recursive_force(s: str) -> bool:
    for seg in re.split(r"[;&|]+", s):
        toks = seg.split()
        if not toks or "rm" not in toks[0]:
            continue
        flags = " ".join(t for t in toks[1:] if t.startswith("-"))
        if re.search(r"(^|\s)-[a-z]*r|--recursive", flags) and \
           re.search(r"(^|\s)-[a-z]*f|--force", flags):
            return True
    return False
```

危险命令则用一张正则表覆盖常见的「一次性毁灭」操作：

```python
_DESTRUCTIVE_RE = re.compile("|".join([
    r"\bmkfs\b", r"\bdd\s+if=", r":\(\)\s*\{",              # 格式化 / 覆盖磁盘 / fork 炸弹
    r"\b(shutdown|reboot|halt|poweroff)\b",
    r"\bformat\s+[a-z]:", r"\bdel\s+/[sfq]\b",
    r"remove-item[^\n]*-recurse",
    r"\bfind\b[^\n]*\s-delete\b", r"\bgit\s+clean\b[^\n]*-[a-z]*[dfx]",
    r"\bshred\b", r"\btruncate\s+-s\s*0",
]), re.I)
```

敏感文件同样用正则，覆盖 `.env`、`.ssh/`、私钥、各类凭据文件。这里用了分隔符和边界断言，尽量避免把普通单词误判：

```python
_SENSITIVE_RE = re.compile(
    r"(^|[\s=<>@/\\\"'`])(\.env(\.[\w.-]+)?|\.ssh/|id_rsa|id_ed25519|"
    r"\.aws/credentials|\.git-credentials|\.netrc|\.npmrc|\.pypirc|"
    r"credentials(\.json)?)(?=$|[\s\"'`);|&><])", re.I)
```

## 审批：把决定权交回用户

检测到风险后，不是直接拒绝，而是**问用户要不要放行**。项目里有个 `confirm` 回调，TUI 模式下弹一个 `[y/N]`：

```python
reasons = classify_command(command)
if reasons:
    allow = (os.environ.get("DEEPSEEK_ALLOW_DANGEROUS", "").strip().lower()
             in ("1", "true", "yes", "on"))
    if not allow:
        tag = ", ".join(reasons)
        if confirm is None:                       # 没有审批通道 = 非交互模式
            return (f"BLOCKED ({tag}): 该命令有风险，请用户手动执行，"
                    "或设置 DEEPSEEK_ALLOW_DANGEROUS=1 覆盖。")
        if not confirm(command):
            return f"BLOCKED ({tag}): the user denied this command."
```

两个关键设计：

- **非交互模式（`--one`）直接拦截**：管道/脚本场景下没人能点确认，`confirm` 是 `None`，于是默认拒绝，而不是默默放行。这是个「fail closed」的选择。
- **留一个显式逃生舱**：`DEEPSEEK_ALLOW_DANGEROUS=1` 用于你明确知道自己在干什么的场景。默认关闭。

{{< callout title="重要：这是启发式防护，不是沙箱" type="danger" >}}
正则黑名单永远可以被绕过（字符串拼接、变量展开、间接执行的脚本……）。它只能挡住「模型无意中踏入危险区」这类常见失误，**挡不住蓄意攻击**。真正需要隔离时，请用容器、虚拟机或受限用户权限来限制进程能触碰的范围。
{{< /callout >}}

## 防外泄：把密钥从子进程环境里摘掉

一个容易被忽略的泄露路径：模型可以让你执行 `env` 或 `printenv`，然后「顺便」读走环境变量里的 API Key。既然工具是模型控制的，就不能假设它不会这么做。

对策是在启动子进程前，**把所有名字里带 `KEY`/`TOKEN`/`SECRET` 等字样的环境变量剔除**：

```python
_ENV_SECRET_RE = re.compile(r"(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|PASSWD)", re.I)

def _child_env() -> dict:
    """Environment for the bash tool with secrets removed (anti-exfiltration)."""
    return {k: v for k, v in os.environ.items() if not _ENV_SECRET_RE.search(k)}
```

这样即便模型执行 `env`，也看不到 `DEEPSEEK_API_KEY` 之类的东西。注意 Agent 自己**仍然能**读到 key（请求模型需要），只是它不继承给被执行的命令。

## 取消与超时：Ctrl+C 到底做了什么

交互式 Agent 里，Ctrl+C 的默认行为是「中断进程」，直接退出程序。但用户往往只是想**打断当前这一轮**，不是想关掉整个会话。

所以这里把 SIGINT 的语义改掉：不抛异常，而是**设置一个取消标志**，由各处轮询：

```python
_cancel = threading.Event()

@contextlib.contextmanager
def cancellable():
    clear_cancel()
    old = signal.getsignal(signal.SIGINT)
    signal.signal(signal.SIGINT, lambda *_: request_cancel())   # 只设置标志
    try:
        yield
    finally:
        signal.signal(signal.SIGINT, old)
        clear_cancel()
```

模型流式请求和命令执行都会检查这个标志。命令执行处每 0.2 秒轮询一次，超时或取消都会**杀掉整棵进程树**：

```python
while True:
    try:
        out, err = proc.communicate(timeout=0.2)   # 短超时轮询
        code = proc.returncode
        break
    except subprocess.TimeoutExpired:
        if _cancel.is_set():
            _kill_tree(proc)
            return "exit_code=130\nstdout:\n(cancelled)\nstderr:\n(cancelled)"
        if time.monotonic() >= deadline:
            _kill_tree(proc)
            return "exit_code=124\nstdout:\n(timed out)\nstderr:\n(timed out)"
```

杀进程树的实现区分平台：

```python
if os.name == "nt":
    subprocess.run(["taskkill", "/F", "/T", "/PID", str(proc.pid)], capture_output=True)
else:
    os.killpg(os.getpgid(proc.pid), signal.SIGKILL)
```

`/T` 表示连同子进程一起杀，`os.killpg` 则对整个进程组下手。

如果你只用 `subprocess.run(cmd, timeout=...)`，会得到一个隐蔽的问题：**超时只终止 bash，它启动的子进程会变成孤儿继续跑**。手动管理进程组 + 轮询 + 杀进程树，才让「取消」真正干净。

退出码也沿用了 Unix 惯例，模型和用户都能看懂：`124` = 超时，`130` = 被中断。

## 小结

给 Agent 执行能力，安全设计应该和主循环一起做，而不是事后补丁。这个项目的做法可以总结为四条：

1. **分类**：把命令分成破坏性 / 敏感文件 / 网络外传三类，命中就停下来；
2. **审批**：默认问用户，非交互模式 fail closed，另留显式的覆盖开关；
3. **隔离**：执行前剥离环境里的密钥，切断最直接的泄露路径；
4. **可控**：Ctrl+C 只取消当前轮，超时与取消都清理整棵进程树。

同时要清醒地认识到它的边界——**这些是启发式护栏，不是沙箱**。它们能显著降低「模型手滑」的代价，但替代不了真正的隔离。

项目地址：[liku-yu/deepseek-agent](https://github.com/liku-yu/deepseek-agent)。

**相关阅读：**
- [从零开始写一个 Agent：一次极简的尝试](/posts/agent-from-scratch/) —— 先搞懂那个核心循环
- [Agent 的工程细节：上下文裁剪、重试、编码与打包](/posts/agent-engineering-details/) —— 让循环真正可用的工程问题
