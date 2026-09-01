---
name: android-log-analyzer
description: 分析 Android logcat 日志与崩溃日志（Java/Kotlin 崩溃、FATAL EXCEPTION、ANR、native crash/tombstone、OOM 等），定位根因到具体类/方法/行号，结合当前项目源码给出修复建议和结构化分析报告。当用户提供日志文件路径、要求分析崩溃/闪退/卡死/ANR/logcat/异常堆栈/tombstone，或反馈"应用挂了/闪退/没反应"需要排查日志时，必须使用本技能。即使用户没有明确说"分析日志"，只要任务涉及从日志中定位 Android 应用问题，也应使用本技能。
---

# Android 日志与崩溃分析

你的目标不是"读懂日志"，而是**回答三个问题**：为什么崩溃/异常（根因）、问题出在哪段代码（定位）、怎么修（修复方案）。日志只是证据，项目源码才是案发现场。

## 工作流程

按以下顺序执行，不要跳步：

### 1. 摸清日志的规模和结构

先用 Bash 快速了解文件，避免盲目 Read 一个几百 MB 的文件：

```bash
wc -l <日志文件>
```

- 文件小于 ~2000 行：直接 Read 通读。
- 文件较大：用 Grep 定位关键标记（见下方的识别模式），拿到行号后用 Read 的 `offset`/`limit` 读取前后各 100~300 行上下文。**崩溃前的日志往往比崩溃堆栈本身更有价值**（记录了触发路径）。
- 用户给了整个目录：先 `ls -lt` 按时间排序，优先看最新的文件，并留意 `tombstone`、`dropbox`、`anr` 等关键目录/文件名。

### 2. 识别问题类型

**用一条 Bash 命令批量匹配所有标记**（不要逐个标记单独 Grep，大文件上每次全量扫描都很贵）：

```bash
grep -nE 'FATAL EXCEPTION|ANR in|Input dispatching timed out|signal SIG(SEGV|ABRT|BUS)|DEBUG.*backtrace|tombstone|OutOfMemoryError|Failed to allocate|StrictMode|Watchdog|Blocked in handler|FORTIFY' <日志文件> | head -50
```

命中后按行号用 Read 的 offset/limit 精读前后 ±100~300 行。同时用一条命令统计 E 级 tag 分布识别噪音（固件常把 Info 流水打成 E 级）：

```bash
grep ' E ' <日志文件> | awk -F' ' '{for(i=1;i<=NF;i++) if ($i=="E") {print $(i+1); break}}' | sort | uniq -c | sort -rn | head -20
```

高频 tag 抽查 2~3 行样本判定为正常流水后，后续所有 grep 用 `grep -v` 排除它们。

各标记对应的问题类型：

| 标记模式 | 问题类型 |
|---|---|
| `FATAL EXCEPTION`、`AndroidRuntime`、`E AndroidRuntime` | Java/Kotlin 未捕获异常（崩溃） |
| `ANR in`、`Input dispatching timed out`、`Broadcast of Intent`、`executing service` | ANR（应用无响应） |
| `signal SIGSEGV`、`signal SIGABRT`、`signal SIGBUS`、`DEBUG.*backtrace`、`tombstone` | native 崩溃（so 层） |
| `OutOfMemoryError`、`Failed to allocate`、`dalvikvm.*OOM` | 内存溢出 |
| `StrictMode`、`DiskReadViolation`、`NetworkOnMainThread` | StrictMode 违规（不致命但暴露问题） |
| `Watchdog`、`Blocked in handler` | 系统/自定义看门狗 |

若一个都没匹配到，说明日志里没有显式崩溃，转为行为级分析：搜索用户描述的异常现象相关关键词（如页面名、功能模块名、错误码），从时间线上找断点。

### 3. 提取关键信息

针对每处问题，提取：

- **时间戳**：问题发生的时刻，用于对齐多台设备/多个日志源。
- **进程与线程**：PID、TID、线程名。主线程（`main`）崩溃直接致死；子线程的 FATAL 同样致死；ANR 要重点看主线程在干什么。
- **堆栈链**：完整提取异常堆栈，包括所有 `Caused by:` 层级。**真正的根因通常是最深的那个 Caused by**，而不是最外层的包装异常（如 `RuntimeException` 包着的 `NullPointerException`）。
- **定位应用帧**：在堆栈中自顶向下找**第一个属于应用包名**（如 `com.dnake.panel`）的帧——这通常是事故的直接责任人。系统帧（`android.*`、`java.*`）只说明调用入口。

### 4. 关联项目源码定位根因

这是本技能的核心步骤，也是和普通"读日志"的区别：

1. 从堆栈中提取应用帧的类名、方法名、行号（如 `com.dnake.panel.iot.MqttClientController.connect(MqttClientController.kt:87)`）。
2. 用 Glob/Grep 在当前项目中找到对应源码文件并 Read，核对行号处的实际代码（注意：日志可能来自旧版本，行号对不上时在文件中搜索方法名）。
3. **读懂上下文再下结论**：看该方法被谁调用、传参从哪来、什么条件下会触发空值/越界/超时。不要只看崩溃那一行就猜。
4. 对 native 崩溃（tombstone）：关注 `signal`、`fault addr`、backtrace 中的 so 名和函数符号。若无符号表，基于涉及的 so 库（如自研 SIP/音视频库）推断功能模块，并检查对应 JNI 调用处的参数与线程安全。
5. 对 ANR：分析主线程堆栈当前阻塞点（网络/磁盘 IO、锁等待、broadcast 队列），若日志附带了 `traces.txt` 内容，对比各线程状态找出持锁者。

### 5. 输出结构化分析报告

使用以下模板（面向开发者，用中文；代码引用保留英文标识符）：

```markdown
# 日志分析报告：<一句话概括问题>

## 概览
| 项目 | 内容 |
|---|---|
| 问题类型 | （崩溃 / ANR / native crash / OOM / 行为异常） |
| 发生时间 | （日志时间戳） |
| 进程/线程 | （进程名 PID，线程名） |
| 直接影响 | （应用闪退 / 功能不可用 / 卡顿等） |

## 根因分析
<异常链解读：从最深层 Caused by 讲起，说明空值/越界/超时是如何产生的。
引用关键堆栈帧和对应源码 file:line，说明代码中哪里的假设不成立。>

## 问题时间线
<崩溃前关键日志事件按时间排列，还原触发路径>

## 修复建议
1. <首选方案：具体改法，必要时给出修改后的代码片段>
2. <备选方案或防御性措施>

## 验证建议
<如何复现、改完后观察哪些日志标记确认修复>
```

## 分析原则

- **区分直接原因和根本原因**。`NullPointerException` 是直接原因；"XX 异步回调早于初始化完成"才是根本原因。报告要对根本原因负责。
- **一行日志不定案**。下结论前尽量在日志中找到第二处佐证（前置警告、重复模式、相关错误码）。
- **日志可能不全**。如果关键时间段缺失或堆栈被截断，明确告诉用户"需要补充 XX 时间段的日志 / 需要带符号表的 so / 需要复现时的完整 logcat"，不要硬猜到底。
- **多处问题按严重度排序**：致死的崩溃 > ANR > 反复出现的非致命异常 > 警告。
