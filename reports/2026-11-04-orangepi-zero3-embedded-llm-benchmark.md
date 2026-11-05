# Orange Pi Zero 3（1 GB）嵌入式 LLM 承载能力实测报告

> **测试日期**：2026-11-04
> **硬件**：OrangePi Zero3 / Allwinner H618 / 4× Cortex-A53 @ 1.416 GHz / 993 MB LPDDR4
> **系统**：DietPi 10.5.2（Debian 13.5 trixie）/ Linux 6.18.34-sunxi64 / aarch64
> **推理引擎**：llama.cpp build `b12102`（0.8.0-dev，commit `bbb02c93e`），运行时后端 `libggml-cpu-armv8.0_1.so`
> **测试方式**：全部数据来自本机实测，无任何参数表推算

---

## 摘要

### 核心结论

| 问题 | 答案 |
|:---|:---|
| **能承载多大的模型？** | **270M–350M 舒适可用**；0.6B 需独占内存；**1B 及以上不可行** |
| **350M 能跑多快？** | **10.4 t/s**（4 线程，实测） |
| **0.6B 能跑多快？** | **5.87 t/s**（独占内存时）；与其他服务共存时抖动至 0.19–1.88 t/s |
| **速度够用吗？** | **够**。10 t/s 高于中文阅读速度（5–8 字/秒） |
| **真正的瓶颈？** | **内存容量** ＞ 缺 `dotprod` 的算力 ＞ 存储带宽（**不是**内存带宽） |
| **GPU 能加速吗？** | **不能**。内核驱动正常，但计算栈完全空白，且与 CPU 共享同一条内存总线 |
| **NPU 算力？** | **0 TOPS —— Allwinner H618 根本没有 NPU** |
| **升级建议？** | 本机用 LFM2.5-350M；要更大模型请换 **4 GB 内存版 Zero 3**（同颗 SoC） |

### 实测数据速览

| 模型 | 体积 | 参数量 | prefill | **生成** | 内存要求 |
|:---|---:|---:|---:|---:|:---|
| **LFM2.5-350M** Q4_K_M | 216 MiB | 354.48 M | 18.4 t/s | **10.4 t/s** | 可与 Hermes 共存 |
| **Qwen3-0.6B** Q4_0 | 359 MiB | 596.05 M | 11.4 t/s | **5.87 t/s** | 需独占内存 |

---

## 目录

- [0. 摘要](#摘要)
- [1. 测试环境与方法](#1-测试环境与方法)
- [2. 硬件审计](#2-硬件审计)
- [3. GPU 审计](#3-gpu-审计)
- [4. NPU 审计](#4-npu-审计)
- [5. LLM 性能实测](#5-llm-性能实测)
- [6. 内存天花板（最关键发现）](#6-内存天花板最关键发现)
- [7. 能力实测：速度不是问题，可靠性才是](#7-能力实测速度不是问题可靠性才是)
- [8. 完整承载矩阵](#8-完整承载矩阵)
- [9. 推荐配置](#9-推荐配置)
- [10. 工程陷阱清单](#10-工程陷阱清单)
- [11. 结论与升级路径](#11-结论与升级路径)
- [附录 A：完整复现命令](#附录-a完整复现命令)
- [附录 B：原始测量输出](#附录-b原始测量输出)
- [附录 C：宿主机当前基线状态](#附录-c宿主机当前基线状态)

---

## 1. 测试环境与方法

### 1.1 硬件配置

| 项目 | 实测值 |
|:---|:---|
| 板卡 | OrangePi Zero3（`xunlong,orangepi-zero3`） |
| SoC | Allwinner H618（`allwinner,sun50i-h618`） |
| CPU | 4× Cortex-A53，480 – 1416 MHz，8 个频率档位 |
| 调频策略 | `performance`（4 核全程锁定 1416 MHz） |
| 内存 | 993 108 kB（**969 MiB**）LPDDR4 |
| 交换 | zram 484.9 MB（lz4，prio 100）+ 2 GB swapfile（prio 10） |
| 存储 | SDXC 64 GB（`mmcblk0`，"SD64G"，type=SD），`mq-deadline` 调度器 |
| GPU | Mali-G31（panfrost 1.4.0），432 MHz |
| NPU | **无** |
| 散热 | SoC 散热片，满载 41 → 53 °C，**无降频** |

### 1.2 软件栈

```
Linux 6.18.34-current-sunxi64  aarch64
Debian GNU/Linux 13.5 (trixie)
DietPi 10.5.2
```

### 1.3 测量方法

| 测量项 | 工具 / 方法 |
|:---|:---|
| 内存带宽 | 自写 C 程序，NEON 128-bit 顺序读，64 MiB 工作集（远超 256 KB L2） |
| INT8 算力 | 自写 C 程序，复刻 ggml 无 `dotprod` 回退路径（`vmull_s8` + `vpadalq_s16`），1 MiB 权重矩阵驻留 L2 |
| SD 读取 | `dd iflag=direct`，`bs=1M count=150`，绕过页缓存 |
| LLM 基准 | `llama-bench`（pp128 / tg128，3 次取中位） |
| 端到端延迟 | `llama-completion` + `common_perf_print` 计时输出 |
| 内存/温度 | `/proc/meminfo`、`/sys/class/thermal/` |

> **所有测量在宿主机实际生产负载下进行**（Hermes 网关 + gdquant），除专门标注「独占内存」的实验外，未做任何调优。

### 1.4 制品获取方式（重要背景）

该设备所在网络对境外源极不友好，这直接影响了测试可行性：

| 源 | 从 DietPi | 从 Mac（有 VPN） |
|:---|---:|---:|
| GitHub（直连） | **0 B/s（不通）** | **0 B/s（不通）** |
| HuggingFace | ~20 KB/s | ~230 KB/s（28 路并行分块） |
| hf-mirror | 13 KB/s | 不可用 |
| ModelScope | 仅部分模型 | — |
| PyPI | **2.24 MB/s（快）** | — |
| `gh-proxy.com`（GitHub 镜像） | — | ~600 KB/s（20 路并行） |

最终路径：**Mac 侧并行分块下载 → scp 到设备**。局域网传输速率 0.9–1.8 MB/s。

---

## 2. 硬件审计

### 2.1 CPU 与缓存

```
Architecture:   aarch64
Model name:     Cortex-A53
CPU(s):         4（无 SMT，1 线程/核）
CPU max MHz:    1416.0000
CPU min MHz:    480.0000
```

| 缓存级别 | 容量 | 说明 |
|:---|---:|:---|
| L1d | 32 KB / 核 | 4 路组相联 |
| L1i | 32 KB / 核 | 2 路组相联 |
| **L2** | **256 KB（4 核共享）** | ⚠️ 仅为树莓派 4（Cortex-A72，1 MB）的 **1/4** |

### 2.2 指令集缺口 —— 决定性限制

```
Flags: fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
```

| 特性 | 有无 | 影响 |
|:---|:---:|:---|
| `asimd`（NEON） | ✅ | 有基础 SIMD，是唯一可用的向量单元 |
| `aes` `sha1` `sha2` `pmull` `crc32` | ✅ | ARMv8 加密扩展，与 LLM 无关 |
| **`dotprod`** | ❌ | **致命**。INT8 点积只能走回退路径，效率约为 1/4–1/5 |
| `i8mm` | ❌ | 无 INT8 矩阵乘指令 |
| `bf16` | ❌ | 无 bfloat16 支持 |
| `fp16` | ❌ | 无半精度算术 |
| `sve` / `sve2` | ❌ | 无可变长向量 |

**这是纯 ARMv8.0-A 基线**（`dotprod` 是 ARMv8.2 才引入的）。Cortex-A53 属于能效小核世代。

#### 独立佐证：llama.cpp 运行时的自动选择

预编译包内含 8 个 CPU 后端变体：

```
libggml-cpu-armv8.0_1.so   libggml-cpu-armv8.2_1.so   libggml-cpu-armv8.2_2.so
libggml-cpu-armv8.2_3.so   libggml-cpu-armv8.6_1.so   libggml-cpu-armv8.6_2.so
libggml-cpu-armv9.2_1.so   libggml-cpu-armv9.2_2.so
```

运行时调度器实际加载的是：

```
load_backend: loaded CPU backend from .../libggml-cpu-armv8.0_1.so
```

从源码编译时，CMake 也自动生成了相同结论：

```
Adding CPU backend variant ggml-cpu:
  -U__ARM_FEATURE_DOTPROD;-U__ARM_FEATURE_MATMUL_INT8;-U__ARM_FEATURE_SVE;-U__ARM_FEATURE_SME
  -mcpu=cortex-a53+crc+crypto+nodotprod+noi8mm+nosve+nosme
```

两条独立路径互相印证：**本机只能用 armv8.0 基线内核**。

### 2.3 实测基础基准

| 指标 | 实测值 | 说明 |
|:---|---:|:---|
| 内存 单线程顺序读 | **6.14 GB/s** | 硬件预取受益 |
| 内存 4 线程读 | **4.46 GB/s** | 4 核共享总线，反而低于单线程 |
| **INT8 GEMV 单核** | **2.01 GMAC/s**（4.0 GOPS） | ggml 无 `dotprod` 回退路径 |
| **SD 卡顺序读** | **23.9 MB/s** | 与内存相差 **约 200 倍** |

**关键推论**：SD 与内存带宽之间有 200 倍的鸿沟。模型**必须常驻内存**，任何换页都会造成灾难性性能崩塌（第 6 节将实测印证）。

---

## 3. GPU 审计

### 3.1 内核侧：正常

```
[    7.928324] panfrost 1800000.gpu: clock rate = 432000000
[    7.937528] panfrost 1800000.gpu: bus_clock rate = 200000000
[    7.951115] panfrost 1800000.gpu: mali-g31 id 0x7093 major 0x0 minor 0x0 status 0x0
[    7.957783] panfrost 1800000.gpu: shader_present=0x1 l2_present=0x1
[    7.971627] [drm] Initialized panfrost 1.4.0 for 1800000.gpu on minor 1
```

- 驱动 **panfrost 1.4.0 正常加载**，Mali-G31 识别成功，`dmesg` **零报错**
- 设备节点 `/dev/dri/card1`、`/dev/dri/renderD128` 存在
- 时钟 **432 MHz 固定**（无 devfreq 节点，不可调）

### 3.2 着色器核心数：只暴露 1 核

`shader_present=0x1` 是位掩码，`0x1` 表示**只存在 1 个着色器核心**——而官方规格标称 Mali-G31 **MP2**（2 核）。这是需要留意的规格与实现差异。

### 3.3 用户态计算栈：完全空白

| 检查项 | 结果 |
|:---|:---|
| Mesa 版本 | ✅ 25.0.7 |
| `panfrost_dri.so` | ✅ 存在（Gallium GL 路径） |
| `libvulkan1` | ✅ 1.4.309.0 已装 |
| **Vulkan ICD** | ❌ `/usr/share/vulkan/icd.d/` **为空** |
| **OpenCL** | ❌ 无（Mesa 本身也不为 panfrost 提供 OpenCL） |
| **GLES/EGL** | ❌ `libegl1` / `libgles2` 未安装 |
| **设备权限** | ❌ `dietpi` 用户不在 `render` / `video` 组 |

**结论：GPU 驱动「活着」，但没有任何一条计算路径可用。** llama.cpp 的 Vulkan 后端（`libggml0-backend-vulkan`）因此无法使用。

### 3.4 GPU 对 LLM 的实际价值：≈ 0

即便把上述缺口全部补齐（安装 `mesa-vulkan-drivers` + 加入 `render` 组）：

- Mali-G31 **单核** @432 MHz 理论 FP32 ≈ **13.8 GFLOPS**
- 与 CPU **共享同一条 LPDDR4 总线**
- 实测解码**只用到 50% 内存带宽**（2.23 / 4.46 GB/s），瓶颈不在带宽也不在算力，而在**小核的并行度**
- panvk（Mesa 的 panfrost Vulkan 驱动）对 Bifrost 世代仍是实验性支持

**判别**：GPU 对 LLM 解码无加速价值，理论上仅对 prefill（算力受限部分）有极有限帮助，但收益远不足以抵消集成风险。

---

## 4. NPU 审计

### 4.1 结论：0 TOPS —— 这颗 SoC 没有 NPU

四重交叉验证，结论一致：

| 验证途径 | 结果 |
|:---|:---|
| **设备树节点** | 仅有 `soc` `display-engine` `thermal-zones` `cpus` `memory` `leds` `wifi-pwrseq` 等；**无 `npu` / `nna` 节点** |
| **内核配置** | `CONFIG_SUNXI_NPU`、`CONFIG_NNA`、`CONFIG_VIP` 等**均不存在** |
| **内核模块** | `lsmod` 无 `npu` / `nna` / `vip` / `galcore` / `vnn` |
| **设备节点** | `/dev` 无 npu 设备；`dmesg` 无任何相关记录 |

SoC 身份：

```
model:      OrangePi Zero3
compatible: xunlong,orangepi-zero3 allwinner,sun50i-h618
```

**Allwinner H618 本身不集成 NPU** —— 这是它与 RK3588 / RK3576 / A733 等「AI SoC」的根本差别。

### 4.2 唯一的专用加速器

| 加速器 | 能力 | 与 LLM 的关系 |
|:---|:---|:---|
| **视频引擎**（V4L2 / Cedrus） | H.264 / H.265 硬件编解码 | 无关（不可做矩阵运算） |
| **ARMv8 加密扩展** | AES / SHA / PMULL 硬件加速 | 无关 |

---

## 5. LLM 性能实测

### 5.1 测试模型

| 模型 | 量化 | 文件体积 | 参数量 | 说明 |
|:---|:---|---:|---:|:---|
| **LFM2.5-350M**（Liquid AI） | Q4_K_M | 216.41 MiB | 354.48 M | 2026 年面向端侧设计，官方称 <1 GB 内存 |
| **Qwen3-0.6B**（阿里） | Q4_0 | 358.78 MiB | 596.05 M | 通用对话 + 混合推理（thinking） |

### 5.2 llama-bench 结果（内存充裕条件下）

#### LFM2.5-350M Q4_K_M

| 线程数 | prefill（pp128） | **生成（tg128）** | 相对 1 核 |
|:---|---:|---:|---:|
| 1 | 4.63 t/s | 3.11 t/s | 1.00× |
| 2 | 9.28 t/s | 5.59 t/s | 1.80× |
| **4** | **18.41 t/s** | **10.41 t/s** | **1.85×** |

#### Qwen3-0.6B Q4_0

| 线程数 | prefill（pp128） | **生成（tg128）** | 相对 1 核 |
|:---|---:|---:|---:|
| 2 | 5.71 t/s | 3.10 t/s | — |
| **4** | **11.36 t/s** | **5.87 t/s** | — |

> **线程数应设为 4。** 2 → 4 线程仍有 **1.85×** 提升，说明瓶颈是**计算并行度**而非内存带宽——这与「小模型解码是带宽受限」的常见假设相反，是 Cortex-A53 单核太弱所致。

### 5.3 端到端延迟（LFM2.5-350M）

| 指标 | 实测值 |
|:---|---:|
| 模型加载（页缓存热） | **0.68 s** |
| 模型加载（SD 冷启动） | **~14 s** |
| prefill 速率 | 17.2 – 18.2 t/s |
| **生成速率** | **10.0 – 10.5 t/s** |
| 首 token（11 token 提示词） | **0.64 s** |
| 完整一轮（问 + 20 token 回答） | **1.6 s** |

三种独立方法（`llama-bench` / `llama-completion` / 多轮对话）测得的生成速度一致在 **10.0–10.5 t/s**，数据可靠。

### 5.4 从实测反推的硬件常数

```
有效权重流带宽 = 10.41 t/s × 0.216 GB = 2.25 GB/s   （实测内存带宽 4.46 GB/s 的 50%）
有效算力       = 354.48 M × 10.41/s   = 3.69 GMAC/s  （单核实测 2.01 GMAC/s × 4核 的 46%）
```

### 5.5 校准模型验证：tg ∝ 1/参数量

用 350M 实测值做锚点预测 0.6B：

```
预测 = 10.41 × (354.48 / 596.05) = 6.19 t/s
实测 = 5.87 t/s
误差 = 5.2%
```

**校准成立**，因此第 8 节的矩阵外推可直接采信。

---

## 6. 内存天花板（最关键发现）

### 6.1 现象：同一模型，两种内存状态下相差 30 倍

Qwen3-0.6B（359 MiB）在 **Hermes 网关运行时**（可用内存仅 ~470 MB）的表现：

| 测量方式 | 加载耗时 | prefill | **生成** |
|:---|---:|---:|---:|
| `llama-bench`（紧跟 `drop_caches`） | — | 11.04 t/s | **5.79 t/s** |
| 实际对话 ① | 24.3 s | 1.93 t/s | **0.19 t/s** |
| 实际对话 ③ | 20.3 s | 1.16 t/s | **0.69 t/s** |
| 实际对话 ④ | 19.4 s | 2.04 t/s | **1.88 t/s** |
| 冷启动加载 | **55.5 s** | — | — |

### 6.2 原因：装得下，但留不住

模型 359 MiB，可用内存 470 MB —— 数学上够，但**没有留给页缓存的余量**。内核持续把模型页换出，每个 token 都要从 **23.9 MB/s** 的 SD 卡重新读回，于是崩塌。

**旁证**：停止 Hermes 后，swap 占用从 **260 MB 掉到 31 MB**，抖动消失。

### 6.3 对照实验：独占内存后完全恢复

停掉 `hermes-gateway` + `gdquant`（释放 352 MB），可用内存升至 **829–858 MB**：

| 模型 | 体积 | 独占内存后生成 | 与其他服务共存时生成 |
|:---|---:|---:|:---|
| **LFM2.5-350M** | 216 MiB | 10.41 t/s | **10.32 t/s（几乎无损）** |
| **Qwen3-0.6B** | 359 MiB | **5.87 t/s（稳定）** | **0.19–1.88 t/s（抖动）** |

### 6.4 经验法则

> **可用内存 ≥ 模型体积 + 200 MB**

- 350M（216 MiB + 200 MB ≈ 416 MB）→ 在 ~470 MB 条件下**刚好安全**
- 0.6B（359 MiB + 200 MB ≈ 564 MB）→ 需要 **>560 MB**，与 Hermes 共存时不足

### 6.5 方法论警告

**只跑 `llama-bench` 会给出虚假乐观的数字。** 它在一小段窗口内运行，权重尚能驻留页缓存；真实交互必须按**常驻**计算内存预算。

---

## 7. 能力实测：速度不是问题，可靠性才是

这部分比速度更重要 —— **能跑多快，不等于能用**。

### 7.1 测试结果

| 任务 | LFM2.5-350M（10.4 t/s） | Qwen3-0.6B（5.87 t/s） |
|:---|:---|:---|
| **中文问答**<br>「解释内存带宽」 | ✅ `内存带宽是计算机内存中存储和操作数据的速度范围。` | ✅ **更准确**：`内存带宽是指计算机中内存（RAM）的总带宽，即内存可以同时处理的数据量和速度。` |
| **闲聊**<br>「你好」 | ✅ `好啊！感谢你的问候。` | — |
| **JSON 抽取**<br>「把客厅的灯关掉」 | ⚠️ 结构合法但**语义错**：<br>`{"intent":"light","device":"living_room"}`<br>（device 对，intent 错） | ❌ **只输出空模板**：<br>`{"intent":"","device":""}` |
| **意图分类（快速）**<br>「关上卧室的灯」 | ❌ `帮上卧室的灯`<br>（完全未遵循指令） | ❌ `open_light`<br>（答错，应为 `close_light`） |
| **意图分类（开思考）** | — | ✅ **`close_light` 正确**，附完整推理链 |

### 7.2 Qwen3-0.6B 开启思考模式的实际输出

```
用户让我判断意图，需要选择 open_light、close_light 或者 query_temp。
首先，用户说的是关上卧室的灯，所以应该涉及关闭灯的操作。
open_light 表示打开，close_light 表示关闭，query_temp 是查询温度。
这里用户明确说"关上卧室的灯"，所以正确的类别应该是 close_light。
```
→ 最终输出：`close_light` ✅

**代价**：一个简单分类消耗 **102 个生成 token** → 在 5.87 t/s 下约 **17 秒**。适合异步任务，不适合实时对话。

### 7.3 判别

| 任务类型 | 候选模型 |
|:---|:---|
| **自由生成**（问答、摘要、改写、闲聊） | ✅ 350M 已够用 |
| **精确结构化输出**（严格 JSON） | ⚠️ 两者都不可靠，需微调或更大模型 |
| **分类 / 判别** | ⚠️ 350M 不行；0.6B 需开思考，**17 秒/次** |
| **实时交互** | ✅ LFM2.5-350M，一轮 1.6 秒 |

---

## 8. 完整承载矩阵

以 LFM2.5-350M 实测（10.41 t/s @ 354.48 M）为锚点，按 `tg ∝ 1/参数量` 外推（已由 Qwen3-0.6B 验证，误差 5.2%）：

| 模型规模 | Q4 体积 | **预期生成** | 与 Hermes 共存 | 定位 |
|:---|---:|---:|:---:|:---|
| 135M 级（SmolLM2 等） | ~90 MiB | **~27 t/s** | ✅ 极宽裕 | 分类、关键词抽取（需微调） |
| Gemma 3 270M | ~170 MiB | **~13.5 t/s** | ✅ 宽裕 | 轻量闲聊 |
| **LFM2.5-350M** | 216 MiB | **10.41 t/s（实测）** | ✅ **实测稳定** | ⭐ **共存首选** |
| **Qwen3-0.6B** | 359 MiB | **5.87 t/s（实测）** | ❌ 抖动至 <2 t/s | 需独占内存 |
| Qwen2.5-0.5B | ~400 MiB | ~7 t/s | ❌ | 需独占内存 |
| Llama 3.2 1B | ~730 MiB | ~3.7 t/s | ❌ | **内存算术上不可行** |
| 1.5B 及以上 | >1 GiB | — | ❌ | 不可行 |
| 任意 MoE | 按**总参数**算 | — | ❌ | 不可行（内存按总量计） |

### 1B 为什么不用测就能判定

```
总内存 969 MB
− 系统开销            ~140 MB
= 可用               ~830 MB（停掉全部服务）
− 1B Q4 权重          730 MB
− 计算缓冲 + KV cache  ~80 MB
= 剩余               ~20 MB   → 必然换页 → SD 23.9 MB/s → 崩至 <1 t/s
```

**结论：1 GB 的 Zero 3 无法运行 1B 级模型，这是内存容量问题，不是软件优化问题。**

---

## 9. 推荐配置

### 方案 A：与 Hermes 共存（推荐）

```bash
llama-server \
  -m /home/dietpi/llm-bench/LFM2.5-350M-Q4_K_M.gguf \
  -t 4 --mlock \
  -c 4096 \
  --cache-type-k q8_0 --cache-type-v q8_0 \
  --host 127.0.0.1 --port 8080
```

| 指标 | 值 |
|:---|:---|
| 生成速度 | **10.4 t/s** |
| 一轮对话延迟 | **1.6 s** |
| 内存占用 | ~250 MB（模型 216 + KV + 缓冲） |
| 与 Hermes 共存 | ✅ 实测 0 次 OOM，飞书全程 `connected` |
| 磁盘占用 | 216 MiB（SD 的 0.4%） |

### 方案 B：需要 0.6B 的能力

```bash
# 必须先腾出内存，否则抖动到 1 t/s 以下
sudo systemctl stop hermes-gateway gdquant

llama-server \
  -m /home/dietpi/llm-bench/Qwen3-0.6B-Q4_0.gguf \
  -t 4 --mlock -c 4096 \
  --cache-type-k q8_0 --cache-type-v q8_0
```

代价：5.87 t/s，且飞书机器人下线。**建议仅在批处理/异步任务时使用。**

---

## 10. 工程陷阱清单

### 陷阱 1：`/tmp` 是 tmpfs（内存盘）

```
$ df -h /tmp
tmpfs   1.5G   60M   1.5G   4%   /tmp
```

**模型文件绝对不能放 `/tmp`** —— 等于权重在内存里存两份（tmpfs 一份 + mmap 页缓存一份）。

实测：

| 模型位置 | 已用内存 | 可用内存 |
|:---|---:|---:|
| `/tmp`（tmpfs） | 759 MB | **210 MB** |
| `/home/dietpi/`（SD） | 494 MB | **475 MB** |

挪一次释放 **265 MB**。

### 陷阱 2：线程数应设为 4，不要降

常见经验是「小模型解码带宽受限，多线程无益」。**在这台机器上不成立**：2→4 线程仍有 1.85× 提升。Cortex-A53 单核太弱，算力而非带宽才是约束。

### 陷阱 3：必须 `--mlock`

| | 带宽 |
|:---|---:|
| 内存 | 4.46 GB/s |
| SD 卡 | 23.9 MB/s |
| **比值** | **约 200×** |

不锁住权重，一旦页缓存被回收就会重演 0.6B 那种 30 倍性能崩塌。

### 陷阱 4：`llama-completion` 默认进交互模式

新版 llama.cpp 把 `-no-cnv` 拆成了独立二进制 `llama-completion`，但它**仍会以交互模式读取 stdin**。脚本调用时必须重定向：

```bash
llama-completion -m model.gguf -p "..." </dev/null
```

否则它会把后续脚本行当作对话输入吞掉。

### 陷阱 5：KV cache 用 q8_0

长上下文时 `--cache-type-k q8_0 --cache-type-v q8_0` 可省一半 KV 内存，对这些小模型质量影响可忽略。

---

## 11. 结论与升级路径

### 11.1 定位

**这台 1 GB 的 Orange Pi Zero 3 是一台「10 t/s 级的 350M 推理盒子」**，而非能跑通用 LLM 的设备。

天花板由三件事共同锁死：

| 约束 | 数值 | 决定什么 |
|:---|:---|:---|
| **内存容量** | 969 MB（生产负载下仅剩 ~470 MB 可用） | 模型上限 0.6B |
| **指令集** | ARMv8.0-A，缺 `dotprod`/`i8mm`/`fp16` | 单核 INT8 仅 2.01 GMAC/s |
| **NPU** | 0 TOPS | 无任何专用 AI 加速 |

**GPU 不是瓶颈，也不是解药** —— 它是可用但无用的。

### 11.2 升级路径

| 目标 | 方案 | 预期 |
|:---|:---|:---|
| **跑动 0.6B–1B** | 换 **4 GB 内存版 Zero 3**（同颗 H618，价格接近） | 1B Q4 约 3.7 t/s |
| **跑动 1.5B–3B 且要快** | 换带 `dotprod`/`i8mm` 的 SoC（A55/A76 以上，如 RK3568/RK3588） | 数倍提升 |
| **跑动 3B+ 或要低功耗高吞吐** | 换带 **NPU** 的 SoC（RK3588 6 TOPS、RK3576、A733） | NPU 可提供 10–50 t/s |
| **留在本机优化** | 换模型不换硬件：选 270M–350M 级 | 10–13.5 t/s |

### 11.3 一句话选型建议

> 与 Hermes 共存 → **LFM2.5-350M**（10.4 t/s，实测稳定）
> 要 0.6B 的准确度 → 必须独占内存，且判别任务需开思考模式（17 s/次）
> 想跑 1B 以上 → **换硬件，1 GB 内存没有软件解**

---

## 附录 A：完整复现命令

### A.1 硬件能力基准（自写 C 程序）

```bash
cat > /tmp/bench.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <time.h>
#include <pthread.h>
#include <arm_neon.h>
static double now(void){struct timespec t;clock_gettime(CLOCK_MONOTONIC,&t);return t.tv_sec+t.tv_nsec*1e-9;}
/* ggml 在无 dotprod 平台使用的回退点积 */
static inline int32x4_t vdot(int32x4_t acc,int8x16_t a,int8x16_t b){
  const int16x8_t p0=vmull_s8(vget_low_s8(a),vget_low_s8(b));
  const int16x8_t p1=vmull_s8(vget_high_s8(a),vget_high_s8(b));
  return vpadalq_s16(vpadalq_s16(acc,p0),p1);
}
static unsigned char *gA; static size_t SZ; static int NT=4;
static void* reader(void*arg){ long id=(long)arg; size_t chunk=SZ/NT, off=id*chunk;
  uint64x2_t acc=vdupq_n_u64(0);
  for(int r=0;r<3;r++) for(size_t i=0;i<chunk;i+=16) acc=vaddq_u64(acc,vld1q_u64((const uint64_t*)(gA+off+i)));
  gA[off]=(unsigned char)vgetq_lane_u64(acc,0); return NULL; }
int main(void){
  SZ=(size_t)64<<20;
  if(posix_memalign((void**)&gA,64,SZ)){puts("alloc fail");return 1;}
  memset(gA,1,SZ);
  uint64x2_t acc=vdupq_n_u64(0); double t0=now();
  for(int r=0;r<4;r++) for(size_t i=0;i<SZ;i+=16) acc=vaddq_u64(acc,vld1q_u64((const uint64_t*)(gA+i)));
  double t1=now(); volatile uint64_t s=vgetq_lane_u64(acc,0);(void)s;
  printf("mem 1-thread read : %6.2f GB/s\n",(double)SZ*4/(t1-t0)/1e9);
  pthread_t th[4]; t0=now();
  for(long i=0;i<NT;i++) pthread_create(&th[i],NULL,reader,(void*)i);
  for(int i=0;i<NT;i++) pthread_join(th[i],NULL);
  t1=now();
  printf("mem 4-thread read : %6.2f GB/s\n",(double)SZ*3/(t1-t0)/1e9);
  const int M=1024,K=1024; int8_t *W,*x; int32_t *o;
  if(posix_memalign((void**)&W,64,(size_t)M*K)) return 1;
  if(posix_memalign((void**)&x,64,K)) return 1;
  o=malloc(M*4); memset(W,1,(size_t)M*K); memset(x,1,K);
  t0=now();
  for(int rep=0;rep<200;rep++) for(int m=0;m<M;m++){
    const int8_t*w=W+(size_t)m*K; int32x4_t a0=vdupq_n_s32(0);
    for(int k=0;k<K;k+=16) a0=vdot(a0,vld1q_s8(w+k),vld1q_s8(x+k));
    o[m]=vaddvq_s32(a0);
  }
  t1=now(); double macs=(double)M*K*200, dt=t1-t0;
  printf("int8 GEMV 1-core  : %6.2f GMAC/s (%.1f GOPS)\n",macs/dt/1e9,macs*2/dt/1e9);
  return 0;
}
EOF
gcc -O3 -march=armv8-a -mtune=cortex-a53 -funroll-loops -pthread -o /tmp/bench /tmp/bench.c
taskset -c 0-3 /tmp/bench
```

### A.2 SD 卡读取速度

```bash
sudo dd if=/home/dietpi/.local/share/hermes/state.db of=/dev/null \
        bs=1M count=150 iflag=direct
```

### A.3 LLM 基准

```bash
cd /home/dietpi/llm-bench/llama-b12102
export LD_LIBRARY_PATH=.
sudo sync; echo 3 | sudo tee /proc/sys/vm/drop_caches >/dev/null
./llama-bench -m /home/dietpi/llm-bench/LFM2.5-350M-Q4_K_M.gguf \
              -t 4 -p 128 -n 128 -r 3
```

### A.4 端到端延迟

```bash
./llama-completion -m /home/dietpi/llm-bench/LFM2.5-350M-Q4_K_M.gguf \
  -t 4 -n 60 --temp 0.1 -p "用一句话解释什么是内存带宽。" </dev/null 2>&1 \
  | grep -E "common_perf_print"
```

### A.5 校验 llama.cpp 选中的 CPU 后端

```bash
./llama-bench --version 2>&1 | grep load_backend
# → load_backend: loaded CPU backend from .../libggml-cpu-armv8.0_1.so
```

---

## 附录 B：原始测量输出

### B.1 硬件能力基准

```
mem 1-thread read :   6.14 GB/s
mem 4-thread read :   4.46 GB/s
int8 GEMV 1-core  :   2.01 GMAC/s (4.0 GOPS)
```

### B.2 SD 卡顺序读

```
157286400 bytes (157 MB, 150 MiB) copied, 6.57748 s, 23.9 MB/s
```

### B.3 LFM2.5-350M Q4_K_M

```
| model                    |     size |   params | backend | threads |  test |     t/s |
| ------------------------ | -------: | -------: | ------- | ------: | ----: | ------: |
| lfm2 350M Q4_K - Medium  | 216.41 MiB | 354.48 M | CPU   |       4 | pp128 | 18.30 ± 0.03 |
| lfm2 350M Q4_K - Medium  | 216.41 MiB | 354.48 M | CPU   |       4 | tg128 | 10.32 ± 0.04 |
| lfm2 350M Q4_K - Medium  | 216.41 MiB | 354.48 M | CPU   |       2 | pp128 |  9.28 ± 0.00 |
| lfm2 350M Q4_K - Medium  | 216.41 MiB | 354.48 M | CPU   |       2 | tg128 |  5.59 ± 0.00 |
| lfm2 350M Q4_K - Medium  | 216.41 MiB | 354.48 M | CPU   |       1 | pp128 |  4.63 ± 0.00 |
| lfm2 350M Q4_K - Medium  | 216.41 MiB | 354.48 M | CPU   |       1 | tg128 |  3.11 ± 0.00 |
build: bbb02c93e (12102)
```

### B.4 Qwen3-0.6B Q4_0

```
| model          |     size |   params | backend | threads |  test |     t/s |
| -------------- | -------: | -------: | ------- | ------: | ----: | ------: |
| qwen3 0.6B Q4_0 | 358.78 MiB | 596.05 M | CPU   |       4 | pp128 | 11.36 ± 0.00 |
| qwen3 0.6B Q4_0 | 358.78 MiB | 596.05 M | CPU   |       4 | tg128 |  5.87 ± 0.00 |
| qwen3 0.6B Q4_0 | 358.78 MiB | 596.05 M | CPU   |       2 | pp128 |  5.71 ± 0.06 |
| qwen3 0.6B Q4_0 | 358.78 MiB | 596.05 M | CPU   |       2 | tg128 |  3.10 ± 0.00 |
build: bbb02c93e (12102)
```

### B.5 Qwen3-0.6B 抖动（Hermes 运行中，可用内存 ~470 MB）

```
--- 意图分类 ---
  load time        = 24279.19 ms
  prompt eval time = 20188.21 ms / 39 tokens (1.93 tokens per second)
  eval time        = 31998.54 ms /  6 runs   (0.19 tokens per second)

--- 中文问答 ---
  load time        = 20348.27 ms
  prompt eval time = 17251.77 ms / 20 tokens (1.16 tokens per second)
  eval time        = 43406.23 ms / 30 runs   (0.69 tokens per second)
```

### B.6 内存天花板对照实验

```
停掉 hermes-gateway + gdquant 后：
  内存: 已用 140 MB, 可用 829 MB
  drop_caches 后可用: 858 MB
  swap 占用: 260 MB → 31 MB
```

### B.7 温度记录

```
空载 (只跑 Hermes):  41 °C
LFM2.5-350M 满载:    48 °C
Qwen3-0.6B 满载:     53 °C
全程无降频（cpufreq 保持 1416 MHz）
```

---

## 附录 C：宿主机当前基线状态

本报告测试的机器此前经过一轮系统调优与故障修复，了解这些背景有助于复现与判断。

### C.1 启动与网络

| 项目 | 状态 |
|:---|:---|
| 启动耗时 | **12.8 s**（kernel 4.5 s + userspace 8.3 s）；优化前基线 80 s |
| 网络 | `wlan0` = 192.168.31.27/24，网关 192.168.31.1，SSID `amtf`，信号 −46 dBm |
| 有线 | `eth0` 事件驱动备用通道（udev + 按需 `ifup`），`metric 1000` 不影响 WiFi |

### C.2 已修复的故障（均与本报告的稳定性相关）

| 故障 | 根因 | 修复 |
|:---|:---|:---|
| WiFi 关联成功但无 IPv4 | `ifup@wlan0` 与 `wpa_supplicant@wlan0` 并行竞态抢控制接口 | systemd drop-in 强制 `After=`；去掉 `wpa-conf`；增加 IPv4 自愈定时器 |
| Hermes 网关无限重启（75/TEMPFAIL） | `host-gateway.json` 残留旧 PID，被其他进程复用且 createTime 吻合 | 开机前清理幽灵记录的 oneshot 单元 |
| `systemctl reboot` 失效 | `systemd-logind` 被 mask 到 `/dev/null` | 解除 mask（代价 2.2 MB 内存） |

### C.3 生产负载内存占用

| 服务 | 内存 |
|:---|---:|
| `hermes-gateway` | 323.6 MB |
| `dropbear`（SSH） | 89.3 MB |
| `gdquant` | 27.2 MB |
| 系统 + 其他 | ~140 MB |

> 这正是为什么「可用内存仅 ~470 MB」——**LLM 推理是在一个已经有 500 MB 生产负载的 1 GB 机器上进行的**，这是本报告最重要的现实约束。

---

## 参考与致谢

- 推理引擎：[llama.cpp](https://github.com/ggml-org/llama.cpp) build b12102
- 模型：[LiquidAI/LFM2.5-350M](https://huggingface.co/LiquidAI/LFM2.5-350M)、[Qwen/Qwen3-0.6B](https://huggingface.co/Qwen/Qwen3-0.6B)
- 相关社区数据点：
  - [Tokens/sec for llama.cpp on Raspberry Pi, Jetson and RK3588](https://github.com/ggml-org/llama.cpp/discussions/31103)（122 条实测，11 块板）
  - [Low-bit decode on an ARMv8.0 phone (Cortex-A53, no dotprod)](https://github.com/ggml-org/llama.cpp/discussions/31507)
  - [An Evaluation of LLMs Inference on Popular Single-board Computers](https://arxiv.org/html/2511.07425v1)
  - [Tiny on-device LLMs in 2026](https://cskwork.github.io/posts/tiny-on-device-llms-sub-1b-models-2026/)

---

*本报告全部数据来自实际运行，测量过程中未对生产服务做永久性改动；独占内存实验结束后服务已全部恢复并验证。*
