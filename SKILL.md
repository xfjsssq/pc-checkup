---
name: pc-checkup
description: Windows PC 健康检查 — 快速体检 / 标准分析 / 深度诊断，三种模式适配不同用户。检查 CPU、GPU、内存、硬盘、后台软件，给出普通人能看懂的结果和优化建议。
disable-model-invocation: true
---

# PC Checkup — Windows 电脑健康检查

你的电脑跑得正常吗？选一个模式，帮你检查并给出看得懂的建议。

---

## 前置：判断显卡类型

在所有模式开始前，先执行以下判断，确定本机显卡情况，后续根据结果动态调整检查项：

1. **检查 `nvidia-smi` 是否可用**：执行 `nvidia-smi --query-gpu=name --format=csv 2>$null`
   - 有输出 → 标记为 NVIDIA 显卡，后续可使用 nvidia-smi 全系列命令
   - 无输出 → 标记为"非 NVIDIA 显卡"
2. **检查 AMD 独显**：`Get-CimInstance Win32_VideoController | Where-Object { $_.Name -like "*AMD*" -and $_.AdapterRAM -gt 1GB }`
   - 有结果 → 标记为 AMD 独显
   - 无结果 → 仅 AMD 核显或纯 Intel
3. **核显检查**：`Get-CimInstance Win32_VideoController` 列出所有设备
   - 没有独显的设备 → 标记为"仅核显"，GPU 相关检查项全部跳过

**注意**：AMD 显卡的温度/功耗/显存占用无法通过系统自带命令直接读取，标准模式和深度模式中相关 GPU 项会标注"AMD 显卡数据建议用 AMD Adrenalin 软件查看"。

---

## 第一步：选模式

告诉用户三个选项，让用户选一个：

```
🟢 快速体检 — 3 分钟，适合只想确认"电脑有没有毛病"的用户
      输出：纯大白话，只说正常/不正常 + 建议关掉什么

🟡 标准检查 — 5 分钟，适合懂一点电脑的用户
      输出：带关键数值 + 简单分析

🔴 深度诊断 — 10~15 分钟，适合玩家/开发者要完整数据
      输出：全指标表格 + 持续监控 + 完整分析报告
```

**在开始任何检查前，先询问用户选择哪个模式。** 不要默认一个模式，必须让用户选。

---

## 第二步：各模式的检查流程

### 🟢 快速体检

**只检查以下 4 项，严禁多余检查：**

1. **电脑运行时间**
   - 命令：从 `Get-CimInstance Win32_OperatingSystem` 的 LastBootUpTime 算起
   - 超过 7 天没重启 → 建议重启
2. **内存使用率**
   - 命令：`Get-CimInstance Win32_OperatingSystem` 算 TotalVisibleMemorySize / FreePhysicalMemory
   - 只判断：占用<60%=正常 / 60~85%=偏高 / >85%=满
3. **C 盘剩余空间**
   - 命令：`Get-CimInstance Win32_LogicalDisk -Filter "DeviceID='C:'"`
   - 只判断：剩余>20%=正常 / 10~20%=偏少 / <10%=快满了
4. **最吃内存的 3 个软件**
   - 命令：`Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 3 ProcessName, WorkingSet64`

**快速体检的输出要求：**
- ❌ 不能出现任何数字指标（不说"57°C"、"71%内存"）
- ❌ 不能出现 GPU 相关数据
- ❌ 不能出现专业术语（P-State、显存、编码器等）
- ✅ 只用：正常 / 偏高 / 需要注意
- ✅ 超过 7 天没重启，建议重启一下
- ✅ 内存偏高时，告诉用户关哪几个软件
- ✅ 建议用这个格式：

```
🟢 快速体检报告

总体判断：[正常 / 基本正常 / 建议处理]

⏱️ 上次重启：[多久之前]（超过7天建议重启一次）
💾 内存：[正常 / 偏高]
📁 C盘：[还剩很多 / 快满了]
📋 建议：如果你觉得电脑慢，可以关掉这几个软件试试：
   • [软件名] — 正在用不少内存
   • [软件名] — 后台开着可能不需要

一句话：你的电脑 [没问题 / 有点小问题，按上面做就行]
```

---

### 🟡 标准检查

**检查以下 8 项：**

1. **系统概要**
   - 电脑型号、OS 版本、运行时间
   - 命令：`Get-CimInstance Win32_ComputerSystem` 和 `Get-CimInstance Win32_OperatingSystem`

2. **CPU 状态**
   - 型号、占用率、温度（如果能读到）
   - 命令：`(Get-CimInstance Win32_Processor).LoadPercentage`

3. **内存状态**
   - 总量、已用、剩余百分比
   - Top 5 进程按内存排序
   - 命令：`Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 5`

4. **C 盘空间**

5. **GPU 状态**
   - **如果检测到 NVIDIA 显卡**（通过 `nvidia-smi` 是否存在判断）：
     - 型号、温度、占用率、显存用量
     - 命令：`nvidia-smi --query-gpu=name,temperature.gpu,power.draw,utilization.gpu,memory.used,memory.total,pstate --format=csv`
     - P-State 简化为中文描述：P2=高性能、P5=中等节能、P8=深度节能、P12=最低功耗
   - **如果检测到 AMD 独显**（通过 `Get-CimInstance Win32_VideoController` 查找 AMD 独显）：
     - 型号、驱动版本、状态
     - 命令：`Get-CimInstance Win32_VideoController | Where-Object { $_.Name -like "*AMD*" -and $_.AdapterRAM -gt 1GB }`
     - AMD 显卡无法通过命令行读取功耗/温度，输出标注"AMD 显卡温度建议用 AMD Adrenalin 软件查看"

6. **谁在用显卡**
   - NVIDIA：`nvidia-smi pmon -c 1`，列出所有持有 GPU 上下文的进程
   - AMD：无法直接读取，跳过此项
   - 只列出有进程名的，标注数量

7. **电池/电源状态**
   - 是否插电、电池剩余
   - 当前电源计划

8. **异常进程检查** — 检查以下已知的耗资源后台软件是否在运行：
   - ArmouryCrate / 奥创中心
   - NVIDIA Overlay（超过 2 个实例才算异常）
   - 多个 Claude 实例
   - Docker Desktop
   - MuMu / 蓝叠等模拟器
   - Steam / Epic 等游戏平台
   - TranslucentTB

**标准检查的输出要求：**
- ✅ 可以出现关键数值（"57°C"、"用了71%"）
- ✅ 可以把 P-State 说成"显卡处于中等节能状态"
- ✅ 列出哪些软件占用了大量资源
- ✅ 每个检查项后给一行简单解释
- ❌ 不要出现：C-State、PCIe 链路、BAR1、编码会话、DWM 合成
- ✅ 建议用这个格式每项：

```
🟡 标准检查报告

📊 总体评分：[🟢 健康 / 🟡 需留意 / 🔴 有问题]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1️⃣ CPU 处理器
   型号：[CPU 型号]（[核心数]核）
   当前占用：[数值]% ✅ 正常
   温度：正常范围

2️⃣ 内存
   总量 [数值]GB，已用 [数值]%，剩余 [数值]GB
   最吃内存的软件：
   • [软件A] — [数值] MB
   • [软件B] — [数值] MB
   如果觉得卡，可以关掉不用的软件

3️⃣ 显卡 ([显卡型号])
   温度：[数值]°C ✅ 正常（超过85°C才算热）
   占用率：[数值]% ✅ 没在拼命跑
   显存：用了 [数值] / 共 [数值]
   状态：中等节能（正常待机状态）

4️⃣ 后台软件提醒
   以下软件正在后台占用资源，不需要可以关：
   • [软件名] — [原因]
   • [软件名] — [原因]

💡 优化建议（按效果排序）
   ⭐ [建议1]
   ⭐ [建议2]
```

---

### 🔴 深度诊断

**覆盖全部检查项，包括持续监控：**

#### 第 1 部分：系统基础信息

| 检查项 | 命令 |
|--------|------|
| 电脑型号、OS 版本 | `Get-CimInstance Win32_ComputerSystem`、`Get-CimInstance Win32_OperatingSystem` |
| 运行时间 | 从 LastBootUpTime 算 |
| 电源计划 | `powercfg /getactivescheme` 并列出所有可用计划 |
| 电池状态 | `Get-CimInstance Win32_Battery`（含 BatteryStatus 含义解释） |
| C 盘空间 | `Get-CimInstance Win32_LogicalDisk -Filter "DeviceID='C:'"` |
| 正在等待重启的更新 | 检查 `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired` |

#### 第 2 部分：CPU

| 检查项 | 命令 |
|--------|------|
| CPU 型号 + 核心/线程 | `Get-CimInstance Win32_Processor` |
| 当前占用率 | Win32_Processor LoadPercentage |
| 温度 | `Get-CimInstance MSAcpi_ThermalZoneTemperature -Namespace root/wmi`（如果支持） |

#### 第 3 部分：GPU

**如果检测到 NVIDIA 显卡：**

| 检查项 | 命令 |
|--------|------|
| 完整 GPU 信息 | `nvidia-smi --query-gpu=name,temperature.gpu,power.draw,utilization.gpu,memory.used,memory.total,pstate,fan.speed --format=csv` |
| P-State 完整含义 | P2=高性能、P5=中等节能、P8=深度节能、P12=最低功耗 |
| 编码器/解码器会话 | `nvidia-smi -q -d ENCODER,DECODER`，如果活跃 >0 则列出 |
| 所有 GPU 进程 | `nvidia-smi pmon -c 1` 完整输出 |
| 功率上限/当前功率 | `nvidia-smi -q` 中提取 Power Readings 部分 |
| 显存使用详情 | FB Memory Usage（重点关注 Used 数值） |
| BAR1 使用情况 | BAR1 Memory Usage — **注意：BAR1 占用高是 WDDM 驱动正常行为，不代表泄漏**，不要标记为异常 |

**如果检测到 AMD 显卡（包括核显和独显）：**

| 检查项 | 命令/方法 |
|--------|-----------|
| 显卡型号列表 | `Get-CimInstance Win32_VideoController \| Where-Object { $_.Name -like "*AMD*" }` |
| 显存 | 从上述命令的 AdapterRAM 字段读取（核显通常为 512MB 共享内存，独显 >1GB 专用显存） |
| 驱动版本 | 从上述命令的 DriverVersion、DriverDate 读取 |
| 温度/功耗 | ⚠️ **无法通过命令行读取**，深度报告中标注"AMD 显卡的温度和功耗数据建议使用 AMD Adrenalin 或 HWiNFO 64 查看" |
| 持续调用检测 | 无法直接检测，标注"建议用 GPUView 或任务管理器→性能→GPU 观察 AMD 显卡占用" |

**如果没有独立显卡（仅核显）：**
- 跳过所有 GPU 检查项
- 深度报告中标注"本机无独立显卡，GPU 部分跳过"

#### 第 4 部分：进程分析

- Top 10 内存占用进程（含完整路径）
- 所有持有 GPU 上下文的进程（`nvidia-smi pmon -c 1` 列出所有进程名。WDDM 下无法获取具体显存数值，用 pmon 的进程列表代替）
- 警告：如果发现以下进程，标记为"需要注意"：
  - ArmouryCrate、NVIDIA Overlay（>2个实例）、Docker、虚拟机、模拟器

#### 第 5 部分：系统服务分析

- 列出所有非微软的三方自启动服务（按资源消耗排序）
- 标记已知高资源消耗服务

#### 第 6 部分：持续监控（3 分钟，每 5 秒采样）

**NVIDIA 显卡**采样以下指标：
- CPU 占用率、GPU 温度、GPU 功耗、GPU 状态、GPU 占用率、内存使用率

**AMD 显卡**：只能采样 CPU 占用率和内存使用率（GPU 数据无法通过命令行读取），在报告中标注"AMD 显卡持续监控数据有限"

```powershell
# 采样 36 次（3 分钟）
1..36 | ForEach-Object {
    Start-Sleep -Seconds 5
    # 记录：CPU温度、GPU温度、GPU功耗、GPU状态、GPU占用率、内存使用率
    # 输出到 CSV 格式
}
```

采样完成后给出：
- 平均值
- 最大值
- 是否有异常波动

#### 第 7 部分：显卡待机功耗分析

**NVIDIA 显卡**：检查是否有进程阻止显卡进入 P8 深待机状态。列出所有 GPU 上下文持有者并分析可疑进程。如果 P-State 为 P5 或更高（P0/P1/P2），标记有哪些进程持有 GPU 上下文不放。

**AMD 显卡**：AMD 没有 P-State 概念，此部分跳过。标注"AMD 显卡待机功耗无法通过命令行分析，建议用 AMD Adrenalin 查看"。

**仅核显**：跳过此项。

**深度诊断的输出要求：**
- ✅ 表格 + 数值 + 完整技术指标
- ✅ 保留所有原始数据
- ✅ P-State 用原始代号（P0-P12），同时附中文解释
- ✅ 给出所有进程列表和完整路径
- ✅ 监控数据带统计摘要
- ❌ 不要过度解释"温度正常"（专家自己会看）
- ✅ 末尾给出结论清单（按严重性排序）

---

## 第三步：输出报告的原则（所有模式通用）

### 语言风格

- **快速体检**：好像跟邻居聊天。"你的电脑不热，挺好的。要是觉得慢，把那个啥啥关了试试。"
- **标准检查**：像朋友帮忙看电脑。"CPU 占用不高，就是内存用了 71%，后台软件有点多。"
- **深度诊断**：像技术文档。"GPU P5 状态，功耗 11.7W，57°C，存在 5 个 NVIDIA Overlay 进程持有上下文。"

### 异常判断标准（所有模式共用）

| 项目 | 正常 | 偏高 | 严重 |
|------|------|------|------|
| GPU 温度 | <75°C | 75~85°C | >85°C |
| CPU 占用（空闲） | <20% | 20~50% | >50% |
| 内存占用 | <60% | 60~85% | >85% |
| C 盘剩余 | >20% | 10~20% | <10% |
| GPU 功耗（笔记本待机） | <15W | 15~25W | >25W |
| GPU 占用率（空闲） | <5% | 5~20% | >20% |

### 模式隔离规则（重要）

- **快速体检不能出现 GPU 相关数据**，就算 GPU 异常也不能说，只说"系统整体正常"
- **标准检查不能出现** C-State、PCIe、BAR1、编码会话计数
- **深度诊断全部可以出现**
- 每个模式的报告格式必须严格匹配对应模板，不要串

### 无法读取数据时的处理

- 如果某个命令失败或无数据，**直接跳过该项**，不要在报告里显示"无法读取"或报错信息
- 快速体检跳过时完全忽略，标准检查可以用"暂未检测"带过，深度诊断可以标注"数据不可用"

### 最后一步

报告输出完成后，询问用户是否需要把报告保存到桌面（存为 `PC-Checkup-报告.txt`）。深度模式下如果用户需要，可以直接把终端输出的完整内容复制保存。
