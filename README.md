# PC Checkup · Windows 电脑健康检查

> 让 AI 帮你的 Windows 电脑做一次体检，然后用人话告诉你：**哪儿不对劲、该关什么、要不要重启。**

一个 [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills)（`SKILL.md` 格式），纯 PowerShell 只读查询，不装软件、不上传数据、不改注册表。

![platform](https://img.shields.io/badge/platform-Windows%2010%20%7C%2011-0078D6?logo=windows&logoColor=white)
![shell](https://img.shields.io/badge/shell-PowerShell%207-5391FE?logo=powershell&logoColor=white)
![format](https://img.shields.io/badge/format-Agent%20Skill-black)
![license](https://img.shields.io/badge/license-MIT-green)

---

## 为什么需要它

Windows 自带的资源监视器信息很全，但**它不告诉你该干嘛**。而大部分"电脑优化"教程要么让你装一堆清理软件，要么让你删系统文件。

这个 Skill 干的事很朴素：跑几条系统自带命令 → 把结果翻译成正常人能看懂的话 → 给出具体可执行的建议。

- 不懂电脑？选「快速体检」，全程只有"正常 / 偏高 / 需要注意"，不出现任何数字和术语
- 懂一点？选「标准检查」，有关键数值 + 每个数字是啥意思
- 玩家 / 开发者？选「深度诊断」，全指标表格 + 3 分钟持续采样 + 原始数据

---

## 三种模式

| 模式 | 耗时 | 检查项 | 输出风格 |
|:--|:--|:--|:--|
| 🟢 **快速体检** | ~3 分钟 | 4 项：运行时长、内存、C 盘、最吃内存的 3 个软件 | 纯大白话，**不出现任何数字指标** |
| 🟡 **标准检查** | ~5 分钟 | 8 项：系统 / CPU / 内存 / C 盘 / GPU / 谁在用显卡 / 电池电源 / 可疑后台软件 | 带关键数值 + 一句人话解释 |
| 🔴 **深度诊断** | 10~15 分钟 | 7 部分：系统、CPU、GPU 全指标、进程、三方服务、3 分钟采样、待机功耗分析 | 技术文档风格，原始数据全留 |

**模式是严格隔离的**：快速体检就算发现显卡温度异常也不会提（因为它不检查显卡）；标准检查不会冒出 BAR1、C-State、PCIe 这类词；只有深度诊断才全量展开。

---

## 它能检查什么

**系统层**

- 电脑型号、Windows 版本、上次重启距今多久（超过 7 天会建议重启）
- 内存总量 / 占用率 / Top N 进程
- C 盘剩余空间
- 电源计划、是否插电、电池状态
- 有没有「更新装完在等重启」

**显卡层**（会先自动识别显卡类型，再决定查什么）

- **NVIDIA**（有 `nvidia-smi`）：温度、功耗、占用率、显存、P-State、风扇、编解码会话、**所有持有 GPU 上下文的进程**
- **AMD 独显**：型号、显存、驱动版本；温度/功耗读不到，会直接告诉你"用 Adrenalin 或 HWiNFO 看"
- **纯核显**：整块 GPU 检查跳过，不会瞎猜

**后台层**

- 已知的吃资源软件是否在跑：奥创中心、NVIDIA Overlay、Docker Desktop、模拟器（MuMu / 蓝叠）、游戏平台、TranslucentTB 等
- 三方自启动服务
- **谁在阻止显卡进入深度节能**（笔记本待机掉电快最常见的原因）

---

## 安装

```bash
git clone https://github.com/xfjsssq/pc-checkup.git
```

然后把整个 `pc-checkup` 文件夹放进你所用的 AI 客户端的 skills 目录。不同客户端目录不同，按各自文档放置即可，例如：

```
~/.claude/skills/pc-checkup/SKILL.md
```

关键是**目录里必须有 `SKILL.md`**，`agents/openai.yaml` 是给支持该格式的客户端读的显示名和默认提示。

> 这个 Skill 声明了 `disable-model-invocation: true`，也就是**不会被 AI 自作主张调用**，必须你手动触发（输入 `/pc-checkup`，或直接说"帮我检查下电脑"）。

---

## 使用

```
你：/pc-checkup
AI：🟢 快速体检  🟡 标准检查  🔴 深度诊断  —— 选一个？
你：快速体检
AI：（跑 4 条命令，30 秒后）
```

一份快速体检长这样：

```
🟢 快速体检报告

总体判断：基本正常

⏱️ 上次重启：9 天前（超过7天建议重启一次）
💾 内存：偏高
📁 C盘：还剩很多
📋 建议：如果你觉得电脑慢，可以关掉这几个软件试试：
   • Chrome — 正在用不少内存
   • 微信 — 后台开着可能不需要

一句话：你的电脑有点小问题，按上面做就行
```

一份标准检查长这样：

```
🟡 标准检查报告

📊 总体评分：🟡 需留意

1️⃣ CPU   型号：AMD Ryzen 7 7840H（8核）  当前占用：12% ✅ 正常
2️⃣ 内存  共 32GB，已用 71%，剩余 9.3GB
   最吃内存：chrome.exe 3.1GB / Code.exe 1.4GB
3️⃣ 显卡（RTX 4060 Laptop）
   温度：57°C ✅ 正常（超过85°C才算热）
   占用率：3% ✅ 没在拼命跑
   显存：用了 1.2 / 8 GB
   状态：中等节能（正常待机状态）
4️⃣ 后台软件提醒
   • ArmouryCrate — 奥创中心，常驻且吃资源
💡 优化建议（按效果排序）
   ⭐ 关掉奥创中心，能少 5% 的常驻内存
   ⭐ 重启一次，释放累积的句柄和缓存
```

检查跑完后它会问你要不要把报告存到桌面（`PC-Checkup-报告.txt`）。

---

## 判定标准

所有模式共用同一套阈值，避免"同一台机器两个模式结论相反"：

| 指标 | 正常 | 偏高 | 严重 |
|:--|:--|:--|:--|
| GPU 温度 | < 75°C | 75~85°C | > 85°C |
| CPU 占用（空闲） | < 20% | 20~50% | > 50% |
| 内存占用 | < 60% | 60~85% | > 85% |
| C 盘剩余 | > 20% | 10~20% | < 10% |
| GPU 功耗（笔记本待机） | < 15W | 15~25W | > 25W |
| GPU 占用率（空闲） | < 5% | 5~20% | > 20% |

---

## 几个刻意的设计

**读不到就跳过，不报错。** 比如某个 WMI 类在你这台机器上不可用，报告里不会出现一行红色报错——快速体检直接忽略，标准检查写"暂未检测"，深度诊断才标注"数据不可用"。

**BAR1 占用高不算异常。** 这是 WDDM 驱动的正常行为，很多"显存泄漏"教程把它当成问题。这个 Skill 明确禁止把它标成异常。

**AMD 显卡不硬编数据。** 命令行拿不到 AMD 的温度和功耗，那就说拿不到，让你去看 Adrenalin，而不是给个编出来的数字。

**P-State 会翻译成人话。** P2=高性能、P5=中等节能、P8=深度节能、P12=最低功耗。

---

## 目录结构

```
pc-checkup/
├── SKILL.md            # 全部检查逻辑、命令、输出模板
└── agents/
    └── openai.yaml     # 显示名、一句话简介、默认提示词
```

只有一个 `SKILL.md`，没有脚本、没有依赖、没有要执行的二进制——所有命令都写在文档里，由 AI 现场调用系统自带工具。

---

## 隐私

- 全部是**本机只读查询**（`Get-CimInstance`、`nvidia-smi`、`powercfg` 等）
- 不联网、不上传、不写注册表、不装软件、不删文件
- 唯一的写操作是你**明确同意**后，把报告存成桌面上的一个 txt

---

## License

[MIT](LICENSE)

---

<details>
<summary><b>English</b></summary>

**PC Checkup** is an Agent Skill that runs a health check on a Windows PC and explains the results in plain language.

Three modes, strictly isolated from each other:

- 🟢 **Quick** (~3 min, 4 checks) — no numbers, no jargon, just "fine / a bit high / needs attention" plus which apps to close
- 🟡 **Standard** (~5 min, 8 checks) — key metrics with a one-line explanation each
- 🔴 **Deep** (10–15 min) — full metric tables, per-process GPU contexts, 3-minute sampling, idle power analysis

It auto-detects the GPU first: full `nvidia-smi` coverage on NVIDIA, honest "not readable from CLI" on AMD, GPU sections skipped entirely on integrated-only machines.

Everything is read-only PowerShell via built-in Windows tooling. No scripts, no dependencies, no network, no registry writes. The only write happens after you explicitly agree to save the report.

Install: drop the `pc-checkup` folder into your AI client's skills directory (it must contain `SKILL.md`). The skill declares `disable-model-invocation: true`, so it is invoked only when you ask for it.

MIT licensed.

</details>
