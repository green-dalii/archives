# Orange Pi Zero 3（1 GB）嵌入式 LLM 深度优化报告

> **测试日期**：2026-11-04 ~ 2026-11-06
> **硬件**：OrangePi Zero3 / Allwinner H618 / 4× Cortex-A53 @ 1.416 GHz / 993 MB LPDDR4
> **系统**：DietPi 10.5.2（Debian 13.5 trixie）/ Linux 6.18.34-sunxi64 / aarch64
> **推理引擎**：llama.cpp build `b12102`（commit `bbb02c93e`），运行时后端 `libggml-cpu-armv8.0_1.so`
> **前作**：[Orange Pi Zero 3 嵌入式 LLM 承载能力实测](2026-11-04-orangepi-zero3-embedded-llm-benchmark.md)

---

## 摘要

本报告是前作的实质性续篇。前作回答了"能跑多大的模型"，本报告回答**"还能压榨出多少"**，并给出一个否定的答案——但沿途发现了三个前作没有的东西：**一个 6 倍速的工具调用模型、一个可用的决策模型、以及这条硬件路线的精确天花板及其根因**。

### 十条核心结论

| # | 结论 | 关键证据 |
|---|---|---|
| 1 | **瓶颈是算力，不是带宽** | Q8_0 体积大 72%，速度只掉 5% |
| 2 | **速度由内核家族决定，不是位宽** | Q6_K(280MB) 比 Q8_0(362MB) 慢 32% |
| 3 | **没有任何引擎能超过 llama.cpp** | 国产 6 个 + 国外 4 个全部证否 |
| 4 | **根因在 A53 微架构** | 2×64-bit NEON 单元 → 4 MAC/cycle 上限 → int8 不比 FP32 快 |
| 5 | **自研引擎上限 1.3–1.5×，需 3–6 人月** | Arm 官方数据：dotprod 差 5.5×，是 ISA 级差距 |
| 6 | **最大单项收益是 prefix cache（15×）** | 多轮对话第 2 轮起 12.2 s → 0.8 s |
| 7 | **prompt 长度是成本主变量** | 每 100 token = 5.4 秒 |
| 8 | **d1-omni-600M 可跑，分类 83%** | 350M 聊天模型只有 20% |
| 9 | **Needle 2 快 6 倍但不懂中文** | 63.3 t/s / 42 MB / 0.58 s，但中文直接崩 |
| 10 | **1512 MHz 是这颗芯片的物理终点** | OPP 存在但被 bin 门控；vdd 上限 1.1 V |

---

## 目录

- [Part 1 · 测试方法与约束](#part-1--测试方法与约束)
- [Part 2 · 性能基线：从实测反推硬件常数](#part-2--性能基线从实测反推硬件常数)
- [Part 3 · 算力受限的判定：量化实验](#part-3--算力受限的判定量化实验)
- [Part 4 · 优化手段穷举（全部实测）](#part-4--优化手段穷举全部实测)
- [Part 5 · 引擎横评：为什么没有赢家](#part-5--引擎横评为什么没有赢家)
- [Part 6 · 自研引擎可行性裁决](#part-6--自研引擎可行性裁决)
- [Part 7 · Decision AI：d1-omni-600M 实测](#part-7--decision-aid1-omni-600m-实测)
- [Part 8 · Needle 2 实测](#part-8--needle-2-实测)
- [Part 9 · 三模型综合对比与选型](#part-9--三模型综合对比与选型)
- [Part 10 · strata 引擎评估](#part-10--strata-引擎评估)
- [Part 11 · 最终压榨清单](#part-11--最终压榨清单)
- [附录 A · 复现命令](#附录-a--复现命令)
- [附录 B · 原始测量数据](#附录-b--原始测量数据)
- [附录 C · 制品获取：绕过网络封锁](#附录-c--制品获取绕过网络封锁)
- [附录 D · 环境快照](#附录-d--环境快照)

---

# Part 1 · 测试方法与约束

## 1.1 硬约束（所有结论的前提）

| 项目 | 值 | 影响 |
|:---|:---|:---|
| SoC | Allwinner H618，4× Cortex-A53 @ **1.416 GHz**（锁定 performance） | 算力天花板 |
| 指令集 | NEON / aes / sha1 / sha2 / pmull / crc32——**无 dotprod / i8mm / bf16 / fp16 / sve** | 无法使用任何现代 int8 加速内核 |
| L2 | **256 KB（4 核共享）** | 仅为 Cortex-A72 的 1/4 |
| 内存 | 969 MiB，生产负载下 **~470 MB 可用** | 模型上限 0.6B |
| NPU | **无（0 TOPS）** | 无专用 AI 加速 |
| GPU | Mali-G31 单核 @432 MHz，**无 Vulkan ICD / OpenCL / GLES** | 完全不可用 |
| 存储 | SD 卡顺序读 **23.9 MB/s** | 冷启动代价高，抖动致命 |

## 1.2 生产负载（这是最真实的测试条件）

本报告全部测量在**宿主机实际生产负载下**进行：

| 服务 | 内存 |
|:---|---:|
| `hermes-gateway`（含 `cron.scheduler` 子进程） | 与 cgroup 合计 **615.7 MB** |
| `gdquant`（A股量化，运行时 139.5 MB） | 189.9 MB |
| `dropbear` | 89.3 MB |

> 这意味着"可用内存 470 MB"不是一个理想上限，而是**一台已经在跑 500 MB 生产负载的 1 GB 机器**的真实余量。本报告遇到的两次"性能异常"（一次 prefill 掉到 8.8 t/s，一次 server 无法启动）**全部由内存压力造成**，而非代码问题。这个教训本身就是结论。

## 1.3 测量口径

| 测量项 | 工具 |
|:---|:---|
| LLM 基准 | `llama-bench`（pp128 / tg128，2–3 次重复） |
| 端到端延迟 | `llama-server` + `slot print_timing` 日志（prompt eval / eval / total） |
| 真实场景 | `/v1/chat/completions`、`/v1/systemone`、`needle.complete()` |
| 内存 | `/proc/<pid>/status` `VmRSS`、`systemd MemoryCurrent` |
| 频率/温度 | `/sys/devices/system/cpu/cpu0/cpufreq/`、`/sys/class/thermal/` |
| 量化对比 | `llama-quantize --allow-requantize`（同源模型，消除参数量混淆） |

## 1.4 本报告使用的模型

| 模型 | 量化 | 体积 | 参数量 | 用途 |
|:---|:---|---:|---:|:---|
| LFM2.5-350M | Q4_K_M | 218.7 MB | 354.48 M | 基线 |
| LFM2.5-350M | **QAD-Q4_0** | **209.2 MB** | 354.48 M | 最终推荐 |
| Qwen3-0.6B | Q4_0 | 364.5 MB | 596.05 M | 第二个锚点 |
| **d1-omni-600M** | **Q4_K_M** | **235.3 MB** | 587 M | 决策模型 |
| **Needle 2** | CQ2 | **13.1 MB** | 45 M | 工具调用 |

---

# Part 2 · 性能基线：从实测反推硬件常数

## 2.1 两个锚点

| 模型 | 参数量 | prefill (pp128) | **decode (tg128)** |
|:---|---:|---:|---:|
| LFM2.5-350M Q4_K_M | 354.48 M | 18.32 t/s | **10.32 t/s** |
| Qwen3-0.6B Q4_0 | 596.05 M | 11.04 t/s | **5.79 t/s** |

## 2.2 反推出的统一公式

对两个不同规模的模型分别计算 `参数量 × 吞吐`：

| 模型 | params × tg | params × pp |
|:---|---:|---:|
| LFM2.5-350M | 354.48 × 10.4 = **3686** | 354.48 × 18.4 = **6522** |
| Qwen3-0.6B | 596.05 × 5.8 = **3457** | 596.05 × 11.0 = **6580** |
| **离散度** | **6%** | **0.9%** |

两个模型给出几乎相同的常数，因此可以写成预测公式：

```
tg(t/s) ≈ 3570 / 参数量(M)      [decode = 3.57 GMAC/s]
pp(t/s) ≈ 6550 / 参数量(M)      [prefill = 6.55 GMAC/s]
pp / tg ≈ 1.83                  ← 常数
```

## 2.3 这个公式的三个用途

1. **预测任意模型**（已验证误差 5.2%）
2. **判断可行性**（比如 LFM3-1.7B → 3570/1730 = **2.1 t/s**，即便内存够也不可用）
3. **识别异常**（任何偏离此公式 10% 以上的测量，都是被外部因素污染了）

## 2.4 参数表外推

| 模型规模 | Q4 体积 | 预测 tg | 可行性 |
|:---|---:|---:|:---|
| LFM2.5-230M | ~120 MB | 15.5 t/s | ✅ |
| **LFM2.5-350M** | 209 MB | **10.5 t/s（实测）** | ✅ |
| Gemma 3 270M | 250 MB | 13.2 t/s | ✅ |
| **Qwen3-0.6B** | 359 MB | **5.9 t/s（实测）** | ⚠️ 需独占内存 |
| Granite 4.0-350M | 237 MB | 10.2 t/s | ✅ |
| Qwen3.5-0.8B | 538 MB | 4.5 t/s | ❌ 超预算 |
| LFM2.5-1.2B | ~700 MB | 3.0 t/s | ❌ |
| **LFM3-1.7B** | 1050 MB | **2.1 t/s** | ❌ 双重不可行 |
| 1.5B+ | >1 GB | — | ❌ |

---

# Part 3 · 算力受限的判定：量化实验

## 3.1 实验设计

要判断"算力受限 vs 带宽受限"，必须在**同一个模型**上改变位宽，消除参数量混淆。用 `llama-quantize --allow-requantize` 从 Q4_K_M 源文件生成 Q4_0 / Q5_K_M / Q6_K / Q8_0。

> 双重量化会损失质量，但**速度特性完全等价**——而这里只测速度。

## 3.2 结果（正序 + 反序两轮，排除热漂移）

| 量化 | 体积 | prefill | **decode** | 权重流 | 占 4.46 GB/s |
|:---|---:|---:|---:|---:|---:|
| **Q4_0** | 210 MB | 18.2 | **10.88** | 2.40 GB/s | 54% |
| **Q8_0** | **362 MB（+72%）** | 17.1 | **10.31（−5%）** | **3.92 GB/s** | **88%** |
| Q4_K_M | 219 MB | 18.1 | 10.18 | 2.36 GB/s | 53% |
| Q5_K_M | 249 MB | 14.7 | **8.91（−18%）** | 2.33 GB/s | 52% |
| Q6_K | 280 MB | 13.7 | **6.96（−36%）** | 2.04 GB/s | 46% |

## 3.3 两个决定性发现

### 发现一：带宽不是瓶颈

**Q8_0 体积大 72%，速度只掉 5%。** 如果是带宽受限，应该掉 42%。同时 Q8_0 的权重流达到 3.92 GB/s（88% 带宽占用）——**它已经快要撞到带宽墙了，所以掉了那 5%**。

### 发现二：速度由内核家族决定，不是体积

这是最反直觉的一条：

> **Q6_K（280 MB）比 Q8_0（362 MB）慢 32%。**

体积解释不了这个顺序。真正原因是**解包内核的开销**：

| 量化 | 数据结构 | NEON 解包代价 |
|:---|:---|:---|
| **Q4_0 / Q8_0** | 固定宽度 2 的幂次子块（32 元素 / fp16 缩放） | **纯移位+掩码**，最便宜 |
| **Q4_K_M** | 超级块 256 + 分层 6-bit 缩放 | 较贵，但 ggml 有 `q4_K × q8_K` 优化路径 |
| **Q5_K_M** | 5-bit——**跨字节非对齐位提取** | 昂贵 |
| **Q6_K** | 6-bit——同样的非对齐位提取 | **最昂贵** |

Cortex-A53 是**顺序双发射小核**，没有乱序执行的延迟隐藏能力，因此这类多字节位操作的开销被完整暴露。而在有 `dotprod` + 乱序的大核上，这个差异会被完全淹没——**这就是为什么通用建议"Q5 只是稍慢"在这里失效**。

## 3.4 直接得到的两个行动项

1. **换 QAD-Q4_0**：比 Q4_K_M 快 `10.88/10.18 = 6.9%`，且体积小 10 MB、质量更高（QAD 回收 97% 量化损失）
2. **永远不要用 Q5_K_M / Q6_K**：在这个架构上它们没有任何理由

---

# Part 4 · 优化手段穷举（全部实测）

## 4.1 llama.cpp 参数扫描 —— 全部无效

基线：LFM2.5-350M QAD-Q4_0，4 线程，pp128 18.6 / tg128 10.56。

| 配置 | pp128 | tg128 | 判定 |
|:---|---:|---:|:---|
| `-t 4`（基线） | 18.6 | **10.56** | 最优 |
| `-t 3` | 13.77 | 7.91 | 明显更差 |
| `-t 4 -C 0x0f`（绑核） | 17.96 | 10.12 | 无收益 |
| `-t 4 -b 64 -ub 32` | 17.63 | 10.40 | 噪声内 |
| `-t 4 --repack 1`（Q4_0） | 9.69 | 5.64 | **反而更慢** |
| `-t 4 --repack 0`（Q4_0） | 10.89 | 5.78 | 更优 |

**`--repack` 这条最常见的 ARM 优化建议在无 dotprod 的 A53 上直接失效。**

### 线程数扩展性（LFM2.5-350M Q4_K_M）

| 线程 | tg | 相对 1 核 |
|---:|---:|---:|
| 1 | 3.11 | 1.00× |
| 2 | 5.59 | 1.80× |
| **4** | **10.32** | **1.85×（累计 3.32×）** |

近线性 → **确认算力受限**（若带宽受限，2 线程就该饱和）。

> **修正记录**：早期曾判断"2–3 线程就带宽饱和"，实测证明是错的。

## 4.2 投机解码 —— 全部是负收益

本机 `--spec-type` 支持 11 种（含 5 种免草稿模型的 n-gram 变体）。在同一 JSON 生成任务上实测：

| spec-type | 生成 t/s | 相对基线 |
|:---|---:|---:|
| none（基线） | **9.8** | — |
| ngram-mod | 9.8 | 持平 |
| ngram-simple | 7.1 | **−28%** |
| ngram-map-k4v | 7.1 | −28% |
| ngram-map-k | 6.7 | **−32%** |
| ngram-cache | 5.1 | **−48%** |

**原因**：理论上投机解码能把解码（46% 效率）转成 prefill（81% 效率），但 4 个慢核上的草稿/校验记账、变长 batch 的图重建开销，超过了批量前向省下的收益。

## 4.3 Prefix cache —— 唯一的数量级收益

### 实验一：完全相同的请求

| 次数 | prompt eval |
|---|---|
| 第 1 次 | **26032 ms / 466 tokens** |
| 第 2–4 次 | **287 ms / 4 tokens** |

**91× 加速。**

### 实验二：真实 agent 模式（多轮追加）

| 轮次 | prompt eval | 说明 |
|---|---|---|
| 第 1 轮 | **12218 ms / 226 tok** | 全量 prefill（含 system prompt） |
| 第 2 轮 | **788 ms / 13 tok** | 只算新增一轮 |
| 第 3 轮 | 1061 ms / 18 tok | — |
| 第 4 轮 | 797 ms / 13 tok | — |

**第 2 轮起 15× 加速。**

### 但有一个反直觉的边界

我最初测出"prefix cache 无效"——因为那组实验每个请求**替换**了末尾问题：

| 场景 | 结果 |
|---|---|
| 完全相同的请求 | ✅ 命中（462/466 token 复用） |
| 多轮**追加**（真实 agent） | ✅ 命中 |
| **替换末尾问题**（同一 system prompt，不同问题） | ❌ **不命中，全额重算** |

**设计含义**：必须用**有状态的长会话**；无状态 API 设计在"共享 system prompt 但不同问题"时不会命中缓存，每次都付全价。

## 4.4 Prompt 长度的真实成本

prefill 吞吐**恒定 18.5 t/s**，与长度无关：

| prompt | prefill 速度 | 耗时 |
|---:|---:|---:|
| 32 tok | 18.50 t/s | 1.7 s |
| 64 | 18.65 | 3.4 s |
| 128 | 18.63 | 6.9 s |
| 256 | 18.63 | 13.7 s |
| 512 | 18.48 | **27.7 s** |
| 1024 | 18.18 | **56.3 s** |

> **每 100 个 prompt token = 5.4 秒。**

对比前作一个月的内核优化只换来 8%——**少写 200 字提示词就省 11 秒**。

> **修正记录**：中途曾测出"1024 token 需 116 秒、prefill 降到 8.8 t/s"。那是可用内存只剩 145 MB 时的**抖动假象**。内存充足时复测为 56.3 秒 / 18.18 t/s。这再次印证内存悬崖。

## 4.5 热降频 —— 不存在

| 状态 | 温度 | tg |
|---|---|---|
| 冷态 | 41 °C | 10.46 t/s |
| 热态 | **68 °C** | **10.53 t/s** |

差 **0.7%**。旁证：
- `cooling_device0 cpufreq-cpu0: cur_state=0`（降频状态 0 = 未介入）
- `scaling_max_freq` 全程 **1416000**
- CPU thermal trip：`60(passive)` `70(passive)` `110(critical)`

**68 °C 距 critical 110 °C 还有 42 °C 余量。散热片够用，前作全部数据有效。**

## 4.6 超频：1512 MHz 存在，但被芯片体质锁死

设备树 OPP 表实测解码（DT 为大端）：

| 频率 | 电压 | `opp-supported-hw` |
|---:|---:|---|
| 1320 MHz | 1.100 V | `0x1d` = bin 0,2,3,4 |
| **1416 MHz** | 1.100 V | `0x2d` = bin 0,**2**,3,5 ← **本机可用** |
| **1512 MHz** | 1.100 V | `0x2a` = bin 1,3,**5** ← **本机被过滤** |

本机 1416 可用而 1512 不可用 → 芯片 speed bin 属于 {0, 2}，而 1512 只对 {1,3,5} 定义了电压。

更硬的限制：

```
vdd-cpu: 当前 1100000 uV | min 500000 | max 1100000
```

**AXP313A 的 DCDC2 上限就是 1.1 V，无法再加压。**

> 要解锁 1512 需要 DT overlay 给 bin 0/2 补上 1.1 V 定义。这是**低体质芯片在未验证电压下运行**——Allwinner 自己就因为"猜电压不可靠"而放弃了更高频率。收益 **+6.8%**（10.5 → 约 11.2 t/s），有 panic 与数据丢失风险。
>
> **判断：不推荐**。第 4.3 和 4.4 节的收益（15× 与每 100 token 5.4 秒）远大于这 6.8%。

## 4.7 输出词表裁剪：还有 18.9% 在白白流失

从 `llama-quantize` 日志确认：`token_embd = [1024, 65536]` → n_embd = 1024, n_vocab = 65536。

```
lm_head = 65536 × 1024 = 67.1 M MAC/token
        = 354.5 M 的 18.9%
```

**每生成一个 token，近 1/5 的算力花在"从 65536 个词里挑一个"上。**

| 裁到 | 理论加速 |
|---|---|
| 8192 词表 | **+19.9%** |
| 4096 词表 | +21.6% |

这是除换硬件外单项收益最大的手段，但需要改模型文件（裁 `output.weight` + 加 `n_vocab_out`），**侵入性强**。

---

# Part 5 · 引擎横评：为什么没有赢家

## 5.1 国产嵌入式引擎：6/6 全灭

| 引擎 | 死的理由 |
|:---|:---|
| **Tengine**（OPEN AI LAB，ARM AI 生态主力） | 无 LLM/Transformer 通路，纯 CNN 时代产物 |
| **Paddle Lite / FastDeploy**（百度） | 已停更，LLM 方向转到 GPU |
| **TNN**（腾讯） | 仅静态 encoder，无自回归 |
| **MegEngine / InferLLM**（旷视） | **自述是移植 llama.cpp** |
| **Arm Compute Library** | 无 LLM 运行时；量化路径走 KleidiAI，而 KleidiAI 只要 DotProd/I8MM/SME |
| **ncnn**（腾讯） | 唯一有 A53 手写 asm + 官方 `benchncnn_llm`，但 int8 只到 **W8A8**（比 W4 又慢又大） |

## 5.2 国际引擎：4/4 不可用

| 引擎 | 死因 |
|:---|:---|
| **MNN**（阿里） | 官方 Wiki 明确：融合低比特 GEMM 内核范围**只有 ARMv8.2 sdot 与 ARMv8.6 i8mm**。其论文的 8.6× prefill / 2.3× decode 测在**骁龙 8 Gen 3**上 |
| **ExecuTorch**（Meta） | KleidiAI 增益明写"**v9 + i8mm 上最高 20%**"，A53 走 XNNPACK 回退 |
| **ONNX Runtime + XNNPACK** | 内核真实存在（`2x8c8-aarch64-neon-mlal-padal-cortex-a53.S`），但**无 W4 LLM 通路** |
| **llamafile / ik_llama.cpp** | 其 aarch64 matmul 优化**早已回流 ggml 主线**，无增量收益 |

## 5.3 根因：Cortex-A53 的 NEON 只有 64-bit 宽

这是整份报告最重要的技术发现。

| 事实 | 后果 | 来源 |
|:---|:---|:---|
| **2 个 NEON/FP 单元，每个只处理 128-bit 寄存器的 64-bit 半** | 一条 128-bit NEON 指令**占满两个单元**，无法与之并发任何 NEON 指令 | [destevez 微基准](https://destevez.net/2025/02/coding-neon-kernels-for-the-cortex-a53/)、[Arm 官方论坛](https://community.arm.com/support-forums/f/dev-platforms-forum/12796/dual-emission-problem-related-to-neon-instruction-set-on-a53) |
| 因此每周期上限 = **4 个 FP32 lane-MAC** | 4 核 @1.416 GHz = 22.6 GMAC/s 屋顶 | 推导 |
| int8 用 `SMULL + SADALP` 模拟：`smull v.8h` 产出 128-bit，占 2 单元周期 → **8 MAC / 2 cycle = 4 MAC/cycle** | **int8 在 A53 上完全不比 FP32 快** | 同上 |
| L1d load 只有 64-bit，store 128-bit；**AGU 只有 1 个** | load 与 store 不可同周期 | 微基准 |
| fmul/fadd 结果延迟 **4 cycle** | 需 4 路独立展开才不 stall | 微基准 |
| **无公开 A53 Software Optimization Guide（NDA）**，`llvm-mca` 的 A53 模型是错的 | 只能靠手写微基准 | Arm 官方回帖 |

### Arm 官方数据：dotprod 是 5.5× 的差距

> int8 点积：**无 dotprod 402 cycle vs 有 SDOT 73 cycle**

这是 **ISA 级差距，手写汇编补不上**。

## 5.4 编译级优化：天花板 3–5%

关键洞察：**ARM 上"编译选项"的收益绝大部分来自打开 ISA 特性（dotprod/i8mm），而不是调优。** 同一台 Neoverse N2 仅改 `-mcpu` 就有 5.75× prefill——因为那打开了 dotprod。

| 选项 | 预期收益 |
|:---|:---|
| `-mcpu=cortex-a53` | 0~3% |
| `-O3` | 0%（ggml Release 默认） |
| `-flto` | 0~2%（ggml op 走函数指针表） |
| `-ffast-math` | 0~2%，有精度风险 |
| **PGO** | 需在 4×A53 上跑 profile 采集 |

**验证成本 40–90 分钟交叉编译，收益 3–5%。不划算。**

## 5.5 GPU：不是解药

| | Mali-G31 单核 | 4× Cortex-A53 |
|---|---|---|
| 频率 | **432 MHz（固定，无 DVFS）** | 1416 MHz |
| 理论 FP32 | 约 7–14 GFLOPS | 约 45–90 GFLOPS |

差距 3–10×，来源是**纯时钟 3.28×** + 入门级 GPU 设计。

但真正的杀手是**算术强度**：

```
LLM 解码 = GEMV（batch=1），算术强度 ≈ 2 FLOP/字节
GPU 打赢 CPU 需要 10–100 FLOP/字节
```

而且 GPU 与 CPU **共享同一条 LPDDR4 总线**，209 MB 权重 ÷ 4.46 GB/s = 47 ms/token = 21 t/s 物理上限——GPU 用同一条总线，不可能突破。

**加上软件栈全空**：无 Vulkan ICD、panfrost 的 rusticl 需要 Mesa 26.1.6（Debian 13 只有 25.0.7）、无 GLES、`dietpi` 不在 render/video 组。且 GPU PLL 不支持动态变频，社区尝试 DVFS 直接导致 panfrost oops。

---

# Part 6 · 自研引擎可行性裁决

## 6.1 现状与屋顶

| 路径 | 现状 | 屋顶 | 现实可达 | 预期收益 |
|:---|---:|---:|---:|---:|
| **Decode** | 3.7 GMAC/s（屋顶 **46%**），带宽用 2.40/4.46 GB/s | 带宽屋顶 ≈7.9 GMAC/s；算力屋顶（含解包 2–2.5 MAC/cyc/core）8–10 GMAC/s | 55–65% ≈ 5.0–5.5 GMAC/s ≈ **14–15 t/s** | **+30~40%** |
| **Prefill** | 6.5 GMAC/s（屋顶 **81%**） | 同上 | 85–90% | **+8~25%** |
| **全局** | — | — | — | **1.2–1.35×（上限 1.5×）** |

## 6.2 为什么 4x4 权重重排在本机无效

4x4 repack 的价值是喂 **SDOT 的 4-byte 分组**（一条 SDOT 吃 16 权重）。无 SDOT 时仍要走 SMULL/SADALP，重排只是把 `zip/ins/tbl` 从热循环挪到打包阶段——而这些 **128-bit 洗牌指令在 A53 上同样吃满两个 NEON 单元**，与 MAC 争同一发射带宽；单 AGU 又让更大访问步长更难流水。

llama.cpp 的 4x4 内核本为 dotprod/i8mm 设计，**armv8.0 上 `--repack` 得 9.69 < 10.89 t/s 是预期行为**。

## 6.3 裁决：技术可行，经济不划算

```
天花板：被「无 SDOT」+「4.46 GB/s 带宽」双重锁死 → 最好 1.3–1.5×
工作量：全手写汇编 + 自研 4-bit 打包格式 + 全部算子（RMSNorm/RoPE/短卷积/GQA）
        + 自研量化工具链 + 正确性验证 = 3–6 人月
先例：   查不到任何为 ARMv8.0 小核手写 LLM 内核并公布性能的项目
```

### 如果仍要投入，唯一的生死实验（2–3 天）

写一个 **L1 驻留的 SMULL+SADALP GEMV 微基准**，测真实 MAC/cycle：

| 结果 | 决策 |
|:---|:---|
| **≥ 3.0 MAC/cycle/core** | 值得投入（说明还有 2× 空间） |
| **< 2.5 MAC/cycle/core** | **直接放弃** |

> 我实测的单核 2.01 GMAC/s 换算为 **1.42 MAC/cycle**。虽然那是含解包的真实内核，但距离 3.0 的门槛仍有距离，**大概率会落在"放弃"一侧**。

### 一个被实测否掉的调研建议

有调研建议"因为单线程带宽 6.14 > 4 线程 4.46 GB/s，decode 也许 2 线程更好"。

**实测直接否掉**：1/2/4 线程 = 3.11 / 5.59 / 10.32 t/s，近线性。因为 Q4_0 解码只用 2.40 GB/s 带宽，瓶颈是算力不是带宽。

---

# Part 7 · Decision AI：d1-omni-600M 实测

## 7.1 这是什么

Liquid AI 的 `d1` 系列是**决策模型**——不生成 token，而是在单次前向里对固定选项集返回**校准概率**。

```
381 M  文本主干（LFM2.5-Encoder-350M，双向注意力）+ 决策头
 94 M  SigLIP2 视觉编码器  ┐ 打包在独立 mmproj 文件
112 M  FastConformer 音频编码器 ┘
```

## 7.2 本机原生支持（无需任何改造）

**llama.cpp b12102 已内置支持**（PR #30114 / #29818）。用错端点会得到明确提示：

```
Error: This model is a decision model without text generation, use /v1/systemone
```

日志确认：`init: decision model type: lfm2-d1-omni`

## 7.3 GGUF 元数据揭示的设计

```
lfm2.decision.type                    = lfm2-d1-omni
lfm2.attention.causal                 = False      ← 双向注意力（encoder，非 decoder）
lfm2.decision.block_count             = 2          ← 2 层决策头
lfm2.decision.temperature.choice.2    = 1.7465     ┐
lfm2.decision.temperature.choice.3_5  = 1.3999     │ 按「问题类型 ×
lfm2.decision.temperature.choice.6_10 = 1.1751     │  选项数量」分档的
lfm2.decision.temperature.choice.11   = 1.3725     │ 温度校准
lfm2.decision.temperature.noul        = 1.0        │
lfm2.decision.temperature.score.3_5   = 1.7301     ┘
```

**这就是它的概率为什么可信**——普通 chat 模型 prompt 出来做不到。

## 7.4 部署

```bash
llama-server -m d1-omni-600M.Q4_K_M.gguf \
  -t 4 -c 1024 -b 512 -ub 512 --cache-ram 0 \
  --host 127.0.0.1 --port 8080
```

```bash
curl http://127.0.0.1:8080/v1/systemone -d '{
 "state":"帮我关上卧室的灯",
 "questions":{"intent":{"type":"choice",
   "instructions":"判断用户意图类别",
   "criteria":{"open_light":"打开或点亮某盏灯",
               "close_light":"关闭或熄灭某盏灯",
               "query_temp":"询问某个房间的温度读数",
               "other":"与灯光和房间温度都无关的请求"}}}}'
```

返回：

```json
{"answers":{"intent":{"type":"choice","choice":"close_light",
  "probabilities":{"open_light":0.040,"close_light":0.924,
                   "query_temp":0.0001,"other":0.036},
  "confidence":0.899}},
 "usage":{"input_tokens":56,"output_tokens":0}}
```

## 7.5 实测结果

### 意图分类准确率

| 输入 | 期望 | 选择 | 置信度 | 耗时 |
|:---|:---|:---|---:|---:|
| 帮我关上卧室的灯 | close_light | **close_light** | 0.899 | 4.44 s |
| 打开客厅的灯 | open_light | **open_light** | 0.993 | 4.04 s |
| 现在卧室几度 | query_temp | **query_temp** | 0.811 | 4.08 s |
| 今天天气怎么样 | other | query_temp ❌ | 0.967 | 3.96 s |
| 把书房的灯关掉 | close_light | **close_light** | 0.806 | 4.74 s |

**4/5。** 优化 criteria 描述（改成完整句子）并加入第 6 个测试后：**5/6（83%）**。

> 唯一的错判「今天天气怎么样」→ `query_temp` 在语义上确实有歧义。而且 "帮我看看明天要不要带伞" → `other` 是正确的。

### 延迟成本结构

| state 长度 | input_tokens | 单问题延迟 |
|:---|---:|---:|
| 短（2 字） | 34 | **2.61 s** |
| 中（16 字） | 49 | 3.74 s |
| 长（80 字） | 100 | **7.74 s** |

**同一 state 问 3 个问题：11.85 s** ≈ 3× 单问题 —— 确认 **N 个问题 = N 次独立前向，不复用前缀**（双向注意力）。

### 内存

| 量化 | 体积 | 与 BF16 一致率 | 本机 |
|:---|---:|---:|:---|
| Q2_K | 163.1 MB | — | ✅ |
| Q3_K_S | 183.6 MB | — | ✅ |
| Q3_K_M | 197.2 MB | — | ✅ |
| Q4_K_S | 224.7 MB | — | ✅ |
| **Q4_K_M** | **235.3 MB** | 91.0% | ✅ **实测 RSS 277–296 MB** |
| Q5_K_M | 266.9 MB | 95.3% | ⚠️ 且内核更慢 |
| Q6_K | 300.4 MB | 97.3% | ⚠️ 内核最慢 |
| Q8_0 | 388.3 MB | 98.5% | ❌ 会 OOM |

**选择 Q4_K_M 的理由**是双重的：它是能装下的量化里**最快**的（Q5_K_M/Q6_K 因内核家族更慢），且内存余量最大。

## 7.6 多模态：支持，但装不下

`d1-omni-600M` 支持文本 + 图像 + 音频。编码器在**独立的 mmproj 文件**里：

| 文件 | 大小 |
|:---|---:|
| `d1-omni-600M.Q4_K_M.gguf` | 235.3 MB |
| **`d1-omni-600M.mmproj-Q8_0.gguf`** | **250.6 MB** |
| `d1-omni-600M.mmproj-f16.gguf` | 420.8 MB |

用法：`llama-server -m ...Q4_K_M.gguf --mmproj ...mmproj-Q8_0.gguf`

| 模式 | 权重 | 估计 RSS | 470 MB 可用 | 停服务后 830 MB |
|:---|---:|---:|:---:|:---:|
| 纯文本（现测） | 235 MB | **296 MB** | ✅ | ✅ |
| + 视觉/音频 | **486 MB** | 550–600 MB | ❌ | ✅ |

> **纯文本不需要 mmproj** —— 这点官方文档没明写，已实测验证。

---

# Part 8 · Needle 2 实测

## 8.1 这是什么

| 项目 | 规格 |
|:---|:---|
| 出品 | Cactus Compute，**Apache-2.0** |
| 参数 | **45 M** |
| 体积 | **单个 13.1 MB 二进制**（模型烘焙在引擎里） |
| 会话内存 | **28 MB** 标称 / 实测 RSS **42 MB** |
| 量化 | CQ2（Cactus Quants，2-bit） |
| 架构 | Simple Attention Network（arXiv:2607.18363）：Hadamard MLP 替代 FFN + GQA + engram KV 记忆 |
| 官方速度 | Pi 5（A76）**500 tok/s** |
| 平台 | ARM64 / x86-64 / ARMv7 / **RISC-V** / WebAssembly；ESP32-P4 都能跑 |

四个关键设计：

1. **字节级语法约束** —— 从 schema 编译出语法，每个 token 都被约束，**输出保证是合法 JSON**
2. **置信度门控** —— 每个回答带校准置信度，设阈值，低于就升级给大模型
3. **工具检索头** —— 声明几百个工具，内置检索每轮只渲染 top-5
4. **内存有界** —— 256 token 滑窗 + 工具作为 KV sink 钉住，对话再长内存也停在 28 MB

## 8.2 部署（绕过 HuggingFace 封锁）

DietPi 上 HF 不通。解法：**Mac 下载 → scp → 本地注入**。

```
python/cactus_needle-2.0.4-py3-none-manylinux2014_aarch64.whl   12.67 MB
needle2.cact                                                    13.10 MB
```

关键发现：

- **`import needle` 不需要 JAX**（JAX 只用于训练/导出）
- 权重烘焙在 **13.63 MB 的 `libneedle.so`** 里
- 原生 C API：

```c
int  needle_init(const char* system_prompt, const char* tools_json,
                 const char* tool_index_path);
int  needle_complete(const char* input, int max_new_tokens,
                     char* out, int out_capacity);
void needle_reset(void);
int  needle_load(const unsigned char* cact, unsigned long long n);
```

部署命令：

```bash
# 解出引擎
python3 -c "import zipfile; open('libneedle.so','wb').write(
    zipfile.ZipFile('cactus_needle-2.0.4-py3-none-manylinux2014_aarch64.whl'
    ).read('needle/libneedle.so'))"
export NEEDLE2_LIB_PATH=/path/to/libneedle.so
```

## 8.3 性能（引擎自报 + 实测）

| 指标 | 实测值 | 对比 LFM2.5-350M |
|:---|---:|---:|
| **decode** | **63.3 t/s** | **6.0×**（10.56） |
| **prefill** | **102.1 t/s** | **5.5×**（18.5） |
| 单次调用延迟 | **0.39 – 0.86 s**（均值 0.58） | 快一个数量级 |
| 进程 RSS | **42 MB** | **省 200–250 MB** |
| 引擎峰值 RAM | 49.9 MB（自报） | — |
| 初始化 | 1.38 s | — |

## 8.4 准确率（英文）

### 需要调用工具时：8/8 全对

```
turn off the light in the bedroom      → turn_off_light {room: bedroom}    conf 0.821
what is the weather like in Dali       → get_weather    {city: Dali}       conf 0.744
please switch on the kitchen light     → turn_on_light  {room: kitchen}    conf 0.205
set the thermostat to 22 degrees       → set_temperature{degrees: 22}      conf 0.996
switch off the study light             → turn_off_light {room: study}      conf 0.109
turn on the porch lamp                 → turn_on_light  {room: porch}      conf 0.119
how hot is it in Kunming right now     → get_weather    {city: Kunming}    conf 0.165
drop the living room temperature to 18 → set_temperature{degrees: 18}      conf 0.414
```

### 不应该调用工具时：2/4 正确

| 输入 | 行为 | 判定 |
|:---|:---|:---|
| thanks, that's all | 不调用 | ✅ |
| open the garage door（无此工具） | 不调用 | ✅ |
| hello there | → `turn_on_light` | ❌ 幻觉 |
| what can you do | → `get_weather` | ❌ 幻觉 |

综合：**10/12 = 83%**。

## 8.5 置信度头是真的有效

| 结果 | 置信度区间 |
|---|---|
| **正确调用** | **0.109 – 0.996** |
| **幻觉调用** | **0.000 – 0.002** |

**设 0.05 阈值就能把所有幻觉滤掉**，而真实的正确调用最低是 0.109。

这比任何 prompt 工程都可靠——**这是决策模型架构相对 chat 模型的本质优势**。

## 8.6 致命缺陷：不支持中文

```python
agent.complete("帮我关掉卧室的灯")
# → UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe5:
#    invalid continuation byte
#    （引擎输出被截断成非法 UTF-8，直接崩溃，不是返回错误）

agent.complete("昆明今天天气怎么样")
# → []  (1.36 s，返回空)
```

**对 Feishu 机器人这个场景，这一条就判死了。**

## 8.7 必须知道的用法坑

**每次独立请求前必须 `agent.reset()`。** 它是 256-token 滑窗会话模型，连续调用会把历史叠加：

| 用法 | 准确率 |
|:---|---:|
| 不 reset（连续调用） | **2/8** |
| **每次 reset** | **8/8** |

> 我第一次测出 2/8 就是这个原因，不是模型不行。这是部署时最容易踩的坑。

---

# Part 9 · 三模型综合对比与选型

## 9.1 主表（全部真机实测）

| | **LFM2.5-350M**<br>QAD-Q4_0 | **d1-omni-600M**<br>Q4_K_M | **Needle 2**<br>CQ2 |
|:---|:---|:---|:---|
| **体积** | 209 MB | 235 MB | **13 MB** |
| **RSS** | 250 MB | 296 MB | **42 MB** |
| **decode** | 10.56 t/s | — （零 token） | **63.3 t/s** |
| **prefill** | 18.5 t/s | ~17 t/s | **102 t/s** |
| **延迟** | 一轮 1.6 s | 2.6–7.7 s | **0.39–0.86 s** |
| **自由生成** | ✅ 好 | ❌ 不会 | ❌ 不会 |
| **意图分类** | ❌ 1/5 | ✅ **83%** | ⚠️ 需工具化改造 |
| **严格 JSON 工具调用** | ❌ 空模板 | — | ✅ **8/8，语法强制合法** |
| **置信度** | ❌ | ✅ 校准 | ✅ **校准（幻觉≈0.000）** |
| **中文** | ✅ | ✅ | ❌ **崩溃** |
| **运行时** | llama.cpp | llama.cpp `/v1/systemone` | 自带引擎 |
| **部署难度** | 低 | 低（b12102 原生支持） | 中（需绕 HF） |

## 9.2 结论：三者互补，不重叠

```
固定英文 system prompt + 英文输入
   ├─ 要输出 JSON 工具调用 ──→ Needle 2      (0.58 s, 42 MB, 语法保证合法)
   ├─ 要分类/路由        ──→ d1-omni-600M (83%, 0 token, 带概率)
   └─ 要自由生成/中文     ──→ LFM2.5-350M  (10.56 t/s)

中文输入 → Needle 2 出局，只能用另外两个
```

## 9.3 内存预算（这是真正的约束）

| 组合 | 总 RSS | 470 MB 可用 |
|:---|---:|:---:|
| 只要 LFM2.5-350M | 250 MB | ✅ |
| 只要 d1-omni-600M | 296 MB | ✅ |
| 只要 Needle 2 | 42 MB | ✅ |
| LFM2.5 + Needle 2 | 292 MB | ✅ |
| **d1 + LFM2.5** | **546 MB** | ❌ |
| d1 + Needle 2 | 338 MB | ✅ |

## 9.4 三种落地架构

### 方案 A：中文优先（推荐给 Feishu 机器人）

```
意图路由 → d1-omni-600M Q4_K_M（83%）
自由生成 → LFM2.5-350M QAD-Q4_0（10.56 t/s）
两个不能同时常驻，按需切换或二选一
```

### 方案 B：英文工具层（性能最优）

```
用户中文输入
   ↓
LFM2.5-350M 翻译/理解为英文意图（慢，但只有这一步）
   ↓
Needle 2 生成严格合法的工具调用 JSON（0.58 s, 42 MB, conf 门控）
   ↓
d1-omni-600M 兜底处理分类/歧义（83%）
```

总内存 250 + 42 + 296 = 588 MB → 需停 Hermes，或去掉其中一个。

### 方案 C：只选一个

**d1-omni-600M** —— 它解决的是实际最痛的问题（分类不可靠 20% → 83%），而自由生成在 agent 场景中用得没那么多。

---

# Part 10 · strata 引擎评估

> 背景：`niko1221/strata` 是 2026-09-24 创建的 C++ 推理引擎，宣传"Qwen3.8-Flash-Next on any consumer hardware"，首次启动按显卡下载对应引擎。用户希望评估能否借鉴其"硬件/模型协同设计"思路。

## 10.1 它确实存在（先澄清一个误判）

调研过程中有 subagent 凭训练知识断言"没有叫 strata 的 40k-star 推理引擎，疑似幻觉产物"。**这是错的**——用认证的 GitHub API 核实：

```json
{"name":"Strata","createdAt":"2026-09-24T16:40:20Z",
 "pushedAt":"2026-11-05T13:54:56Z","stargazerCount":40424,
 "forks":4416,"open_issues":978,"size_kb":88605,
 "language":"C++","license":"MIT"}
```

那个 subagent 的知识截止时间早于它。**教训：训练知识不能作为"不存在"的证据。**

## 10.2 但它的协同设计全是 GPU 专属

| 目录 | 面向 |
|:---|:---|
| `metal/` | Apple Silicon GPU |
| `sycl/` | Intel GPU |
| `spark/` + NVFP4 | NVIDIA |
| ROCm 资产 | AMD Radeon |

**没有任何 `linux-aarch64` 资产，没有 ARM CPU 后端。** 它支持的模型是 DeepSeek 级别（191–274 GB），对 969 MB 内存毫无意义。README 里所有 benchmark 都是 GPU：

| 硬件 | 输出 | prompt |
|:---|---:|---:|
| RTX 5070 12 GB | 4–6 tok/s | ~110 tok/s (4K) |
| RTX 3060 12 GB | 9–11 tok/s | ~155 tok/s (4K) |
| 1× Radeon R9700 | 21–25 tok/s | 320–360 (8K) |
| 4× Radeon R9700 | 74–77 tok/s | 400–450 (8K) |
| Strix Halo 128 GB | 19–21 tok/s (code) | 87 (4K) |

## 10.3 它的"协同设计"是什么

```
CMakeLists.txt   211 KB   ← per-architecture 编译矩阵
setup.py         707 KB   ← 模型转换规格表
首次启动          按显卡下载对应引擎二进制
```

本质是**把 per-architecture 编译矩阵工业化**，而不是发明了新算法。它运行的是第三方 GGUF 量化权重，且明确推荐外部量化供应商（ISTA-DASLab / Unsloth / Gyro）。

## 10.4 可信度的结构信号分析

| 信号 | 数值 | 判读 |
|:---|---:|:---|
| star 增速 | **963/天**（42 天到 40k） | 异常。llama.cpp 需数年 |
| fork/star | 10.9% | 正常偏低，不构成造假证据 |
| **open issues** | **978（23.3/天）** | **刷 star 不会刷 issue** → 真实使用压力 |
| README 语言 | **8 种** | 营销预算信号 |
| 分发形态 | 桌面 app 安装包 + 首启下载引擎 | 消费级产品，非可复现内核 |

**结论：项目为真、装机量大，但形态是"包装精美的消费级分发产品"，40k stars 的营销成分高于内核工程成分。**

## 10.5 可迁移的方法论：只有一条，而我们已经自己证完了

strata 真正值得学的方法论只有一条：

> **"按设备选内核" + "速度由内核家族决定，不是由体积决定"**

而这一条，本报告 Part 3 已经用自己的实测独立证出来了：

```
Q4_0 10.88 > Q8_0(大72%) 10.31 > Q4_K_M 10.18 > Q5_K_M 8.91 > Q6_K 6.96 t/s
```

**strata 没有给出任何我们没试过的新手段。对本机的直接价值为零。**

---

# Part 11 · 最终压榨清单

## 11.1 按性价比排序

| 排名 | 手段 | 实测/核算收益 | 成本 | 状态 |
|:---|:---|:---|:---|:---|
| **1** | **多轮 prefix cache** | **15×**（12.2 s → 0.8 s） | 零 | ✅ 实测确认 |
| **2** | **缩短 system prompt** | 每 100 token = **5.4 s** | 零 | ✅ 实测确认 |
| **3** | 用 Needle 2 替代工具调用生成 | **6× 速度 + 5× 省内存** | 中（无中文） | ✅ 实测确认 |
| **4** | 用 d1 替代分类 | **20% → 83%** | 中（296 MB） | ✅ 实测确认 |
| **5** | 换 `QAD-Q4_0` | **+6.9%** + 质量 | 已落地 | ✅ 已完成 |
| 6 | 输出词表裁剪（65536→8192） | **+19.9%** | 改模型，侵入强 | 📐 纸面核算 |
| 7 | DT 解锁 1512 MHz | +6.8% | 改 DTB，panic 风险 | ⚠️ 不推荐 |
| 8 | 交叉编译 + PGO | +3~5% | 40–90 分钟 | 不划算 |
| ❌ | 换引擎（国产 6 + 国外 4） | **0%** | — | 已证否 |
| ❌ | 三值推理 | 负 | — | Issue #411 未修 |
| ❌ | 投机解码（5 种变体） | −28~48% | — | 已实测证否 |
| ❌ | GPU / DDR 超频 | 0% | — | GPU 慢 3–10×，DDR 写死 bootloader |
| ❌ | 自研引擎 | 1.3–1.5× / 3–6 人月 | 极大 | 不划算 |

## 11.2 三条使用铁律

1. **一个用户一个长会话**，让 KV 前缀累积 → 第 2 轮起 15× 加速
2. **system prompt 越短越好** → 100 token = 5.4 秒
3. **不要用无状态 API 设计** → 独立请求之间不共享前缀缓存，每次都付全价

## 11.3 推荐配置

```bash
# 自由生成（与 Hermes 共存）
llama-server -m LFM2.5-350M-QAD-Q4_0.gguf \
  -t 4 -c 4096 -b 256 -ub 256 --mlock \
  --cache-type-k q8_0 --cache-type-v q8_0 \
  --host 127.0.0.1 --port 8080

# 决策/分类（需腾内存）
llama-server -m d1-omni-600M.Q4_K_M.gguf \
  -t 4 -c 1024 -b 512 -ub 512 --cache-ram 0 \
  --host 127.0.0.1 --port 8081

# 工具调用（英文，极轻量）
export NEEDLE2_LIB_PATH=/home/dietpi/.cache/cactus-needle/v2/2.0.4/libneedle.so
# Python: from needle import Needle; a=Needle(tools=[...]); a.reset(); a.complete(q)
```

## 11.4 硬件的最终判断

这台机器的天花板由三件事共同锁死，**软件层面已无剩余空间**：

| 约束 | 数值 | 决定什么 |
|:---|:---|:---|
| **内存容量** | 969 MB（生产负载下 470 MB 可用） | 模型上限 0.6B |
| **指令集** | ARMv8.0-A，2×64-bit NEON，4 MAC/cycle | 单核 INT8 仅 2.01 GMAC/s |
| **加速器** | NPU 0 TOPS，GPU 不可用 | 无任何专用加速 |

想要质变只有换硬件：

| 目标 | 方案 | 预期 |
|:---|:---|:---|
| 跑 0.6B–1B | **4 GB 内存版 Zero 3**（同颗 H618） | 1B Q4 约 3.7 t/s |
| 1.5B–3B 且要快 | 带 `dotprod`/`i8mm` 的 SoC（RK3568 / RK3588） | 数倍提升 |
| 3B+ 或低功耗高吞吐 | 带 **NPU** 的 SoC（RK3588 6 TOPS / RK3576 / A733） | NPU 可提供 10–50 t/s |

---

# 附录 A · 复现命令

## A.1 量化位宽对比（消除参数量混淆）

```bash
cd /home/dietpi/llm-bench/llama-b12102
export LD_LIBRARY_PATH=.
SRC=/home/dietpi/llm-bench/LFM2.5-350M-Q4_K_M.gguf

for q in Q4_0 Q5_K_M Q6_K Q8_0; do
  ./llama-quantize --allow-requantize "$SRC" "/home/dietpi/llm-bench/_c-$q.gguf" "$q"
done

for q in Q4_0 Q4_K_M Q5_K_M Q6_K Q8_0; do
  M=/home/dietpi/llm-bench/_c-$q.gguf
  [ "$q" = "Q4_K_M" ] && M=$SRC
  echo -n "$q: "
  ./llama-bench -m "$M" -t 4 -p 128 -n 128 -r 2 2>/dev/null | grep -E 'pp128|tg128'
done
```

> `--allow-requantize` 是必需的，否则对已量化模型会报 `requantizing from type q6_K is disabled`。

## A.2 Prefix cache 验证

```bash
PORT=8100
./llama-server -m LFM2.5-350M-QAD-Q4_0.gguf \
  -t 4 -c 1024 -b 256 -ub 256 --host 127.0.0.1 --port $PORT >/tmp/pc.log 2>&1 &

# 多轮追加模式（真实 agent）：第 1 轮全量，第 2 轮起只算新增
for i in 1 2 3 4; do
  curl -s -o /dev/null http://127.0.0.1:$PORT/v1/chat/completions \
    -H 'Content-Type: application/json' -d @t$i.json
  sleep 1
  grep -oE 'prompt eval time = *[0-9.]+ ms / *[0-9]+ tokens' /tmp/pc.log | tail -1
done
```

预期：`12218 ms/226 tok` → `788 ms/13 tok`。

## A.3 投机解码测试

```bash
# 只有 llama-cli / llama-server 支持 --spec-type，llama-completion 不支持
for cfg in none ngram-simple ngram-cache ngram-mod ngram-map-k ngram-map-k4v; do
  timeout 400 ./llama-cli -m "$L" -t 4 -n 160 --temp 0 --top-k 1 \
    -p "$P" --spec-type "$cfg" </dev/null 2>&1 \
    | grep -oE 'Generation: [0-9.]+ t/s'
done
```

## A.4 d1-omni-600M 部署

```bash
./llama-server -m d1-omni-600M.Q4_K_M.gguf \
  -t 4 -c 1024 -b 512 -ub 512 --cache-ram 0 \
  --host 127.0.0.1 --port 8080

curl http://127.0.0.1:8080/v1/systemone -H 'Content-Type: application/json' -d '{
 "state":"帮我关上卧室的灯",
 "questions":{"intent":{"type":"choice","instructions":"判断用户意图类别",
   "criteria":{"open_light":"打开或点亮某盏灯","close_light":"关闭或熄灭某盏灯",
               "query_temp":"询问某个房间的温度读数",
               "other":"与灯光和房间温度都无关的请求"}}}}'
```

> `--cache-ram 0` 必须加，否则默认 8 GB 缓存配置会失败。

## A.5 Needle 2 部署

```bash
# 1) 在能访问 HF 的机器上下载
BASE=https://huggingface.co/Cactus-Compute/needle2/resolve/main
curl -L -o cactus_needle.whl  "$BASE/python/cactus_needle-2.0.4-py3-none-manylinux2014_aarch64.whl"
curl -L -o needle2.cact       "$BASE/needle2.cact"

# 2) scp 到目标机

# 3) 安装 + 注入引擎
python3 -m venv ~/needle-venv
~/needle-venv/bin/pip install 'cactus-needle<3'
python3 -c "import zipfile
open('libneedle.so','wb').write(
  zipfile.ZipFile('cactus_needle.whl').read('needle/libneedle.so'))"
export NEEDLE2_LIB_PATH=$PWD/libneedle.so

# 4) 使用（注意 reset！）
~/needle-venv/bin/python -c "
from needle import Needle
a = Needle(tools=[{'name':'turn_off_light','description':'Turn off a light',
                   'parameters':{'room':{'type':'string','description':'Room'}}}])
a.reset()                      # ← 每次独立请求前必须 reset
r = a.complete('turn off the bedroom light', max_new_tokens=64)
print(r['function_calls'], r['confidence'], r['decode_tps'])
"
```

> `pip install 'cactus-needle<3'` 装的是 Needle **2**（v2.0.15）。`cactus-needle` 3.x 已改为 **Needle 3**（SimpleAttentionNetwork），两者 checkpoint 格式不兼容。

## A.6 OPP 表解码（DT 为大端）

```bash
python3 - <<'PY'
import os, struct
base = "/proc/device-tree/opp-table-cpu"
for n in sorted(os.listdir(base)):
    d = os.path.join(base, n)
    if not os.path.isdir(d) or not n.startswith("opp-"): continue
    hz = struct.unpack(">Q", open(d+"/opp-hz","rb").read(8))[0]
    hw = open(d+"/opp-supported-hw","rb").read(4)
    print(f"{hz/1e6:7.0f} MHz  supported-hw=0x{struct.unpack('>I',hw)[0]:02x}")
PY
```

---

# 附录 B · 原始测量数据

## B.1 量化位宽曲线（同一模型，正反序两轮）

```
量化        MB | pp正序 pp反序 | tg正序 tg反序 | tg均值
Q4_0        210 |   17.73   18.64 |   10.81   10.95 |   10.88
Q4_K_M      219 |   17.77   18.34 |   10.18   10.19 |   10.18
Q5_K_M      249 |   14.56   14.91 |    8.88    8.95 |    8.91
Q6_K        280 |   13.64   13.75 |    6.97    6.96 |    6.96
Q8_0        362 |   17.41   16.85 |   10.30   10.32 |   10.31
结束 61°C | 频率 1416000 kHz（满载标称 1416000，无降频）
```

## B.2 冷热态对照

```
[冷态] 41°C → tg = 10.46 t/s (max_freq 1416000)
[热态] 67°C → tg = 10.53 t/s (max_freq 1416000)
再次   68°C → tg = 10.53 t/s
→ 冷热差异 +0.7%，无降频
cooling_device0 cpufreq-cpu0: cur=0/max=7
```

## B.3 Prefix cache 原始日志

```
=== 完全相同请求 (466 token) ===
第1次: prompt eval time = 26032.33 ms / 466 tokens
第2次: prompt eval time =   287.40 ms /   4 tokens
第3次: prompt eval time =   294.20 ms /   4 tokens
第4次: prompt eval time =   286.60 ms /   4 tokens

=== raw /completion (264 token) ===
第1次: prompt eval time = 14311.63 ms / 264 tokens
第2次: prompt eval time =   273.77 ms /   4 tokens
第3次: prompt eval time =   275.69 ms /   4 tokens

=== 多轮追加 (真实 agent) ===
第1轮: prompt eval time = 12218.38 ms / 226 tokens
第2轮: prompt eval time =   788.35 ms /  13 tokens
第3轮: prompt eval time =  1060.92 ms /  18 tokens
第4轮: prompt eval time =   797.21 ms /  13 tokens
```

## B.4 投机解码

```
spec-type            生成 t/s
none                      9.8
ngram-mod                 9.8
ngram-simple              7.1
ngram-map-k4v             7.1
ngram-map-k               6.7
ngram-cache               5.1
```

## B.5 d1-omni-600M

```
探测器 (32 token):  {"noul":0.8011}  HTTP 200
引擎识别:          init: decision model type: lfm2-d1-omni
                   decision model reads the embeddings output, enabling embedding mode
RSS:               277 → 285 → 296 MB

意图分类 (原始 criteria):  4/5
意图分类 (优化 criteria):  5/6

延迟: 34tok 2.61s | 49tok 3.74s | 100tok 7.74s
3 问题合并请求: 11.85s (input_tokens=147, output_tokens=0)
```

## B.6 Needle 2

```
引擎自报 (12 次调用平均):
  decode   63.3 t/s
  prefill 102.1 t/s
  peak_ram 49.9 MB

进程 RSS: 42 MB
初始化:   1.38 s (含引擎加载)
单次延迟: 0.39 – 0.86 s

第一批 (优化 criteria 前) 样本:
  decode 57.8 / prefill 80.4 t/s
  decode 56.7 / prefill 84.3 t/s
```

## B.7 OPP 表

```
频率MHz   电压V    各 speed-bin 电压              supported-hw
    480   0.900   base:0.900                     0x3f = 00111111
    600   0.900   base:0.900                     0x12 = 00010010
    720   0.900   base:0.900                     0x2d = 00101101
    792   0.900   speed1:0.900,speed4:0.940      0x12 = 00010010
    936   0.900   base:0.900                     0x0d = 00001101
   1008   0.940   speed0:0.950,speed1:0.940,...  0x3f = 00111111
   1032   0.900   base:0.900                     0x20 = 00100000
   1104   1.000   speed0:1.000,speed2:1.000,...  0x2d = 00101101
   1200   1.020   speed0:1.050,speed1:1.020,...  0x3f = 00111111
   1320   1.100   base:1.100                     0x1d = 00011101
   1416   1.100   base:1.100                     0x2d = 00101101  ← 本机
   1512   1.100   speed1:1.100,speed3:1.100,
                  speed5:1.160                   0x2a = 00101010  ← 被过滤
```

---

# 附录 C · 制品获取：绕过网络封锁

本设备所在网络对境外源极不友好，这是所有实验的前置障碍。

## C.1 各源可达性

| 源 | 从 DietPi | 从 Mac（有 VPN） |
|:---|---:|---:|
| GitHub（直连） | **0 B/s** | **0 B/s** |
| HuggingFace | ~20 KB/s | ~230 KB/s（并行分块） |
| hf-mirror | 13 KB/s | — |
| PyPI | **2.24 MB/s（快）** | — |
| `gh-proxy.com`（GitHub 镜像） | — | ~600 KB/s（20 路并行） |

## C.2 三种绕行策略

| 场景 | 策略 |
|:---|:---|
| **PyPI 包**（cactus-needle） | ✅ 直接 `pip install`（树下快） |
| **GitHub 仓库** | Mac 用 `gh` CLI / `gh-proxy.com` |
| **HuggingFace 模型/引擎** | **Mac 并行分块下载 → scp** |
| **llama.cpp 二进制** | `gh-proxy.com` 前缀（GitHub 直连不通） |

## C.3 并行分块下载器（Mac 侧）

关键设计（踩过的坑）：
- **零填充编号**（`p00001` 而非 `p1`）—— 否则 `cat p*` 会按字典序拼接导致文件损坏
- 每块独立校验大小
- 多轮重试（网络不稳时必需）
- 检测 `Range` 被忽略的情况

> d1-omni-600M 第一次下载时第 3 块卡死（8 轮重试未补全），改用 `curl -C -` 顺序断点续传后成功。**大文件优先用顺序续传而非并行分块**——分块一旦某块卡住，整轮重试都可能无效。

## C.4 传输速率参考

| 操作 | 大小 | 耗时 | 速率 |
|:---|---:|---:|---:|
| LFM2.5-350M-Q4_K_M | 219 MB | 4m13s | 0.86 MB/s |
| Qwen3-0.6B-Q4_0 | 365 MB | 3m18s | 1.84 MB/s |
| LFM2.5-350M-QAD-Q4_0 | 209 MB | 2m31s | 1.38 MB/s |
| d1-omni-600M-Q4_K_M | 235 MB | 2m19s | 1.69 MB/s |
| Needle 2 引擎 + 权重 | 26 MB | <10s | — |

---

# 附录 D · 环境快照

## D.1 硬件

```
SoC:      xunlong,orangepi-zero3 / allwinner,sun50i-h618
CPU:      4× Cortex-A53, 480–1416 MHz, governor=performance（锁定 1416）
L1d/L1i:  32 KB / 32 KB per core
L2:       256 KB（4 核共享）
RAM:      993108 kB (969 MiB) LPDDR4
Swap:     zram 484.9 MB (lz4, prio 100) + /var/swap 2 GB (prio 10)
Storage:  SDXC 63.9 GB, mq-deadline, 顺序读 23.9 MB/s
GPU:      Mali-G31, panfrost 1.4.0, shader_present=0x1, 432 MHz 固定
NPU:      无
Thermal:  cpu 60(passive) 70(passive) 110(critical); gpu/ve/ddr 110(critical)
```

## D.2 软件

```
Linux 6.18.34-current-sunxi64  aarch64
Debian GNU/Linux 13.5 (trixie)
DietPi 10.5.2
Python 3.13.5 / pip 26.0.1
llama.cpp build b12102 (commit bbb02c93e), 后端 libggml-cpu-armv8.0_1.so
```

## D.3 生产服务

| 服务 | cgroup 内存 | 说明 |
|:---|---:|:---|
| `hermes-gateway` | **615.7 MB** | 含 `cron.scheduler --external-worker` 子进程 |
| `gdquant` | 189.9 MB | A股量化，运行时 139.5 MB |
| `dropbear` | 89.3 MB | SSH |
| 系统 + 其他 | ~140 MB | — |

## D.4 制品清单

**设备端** `/home/dietpi/llm-bench/`：

```
llama-b12102/                         llama.cpp 全部二进制 + 8 个 CPU 后端变体
LFM2.5-350M-Q4_K_M.gguf               218.7 MB
LFM2.5-350M-QAD-Q4_0.gguf             209.2 MB   ← 推荐
Qwen3-0.6B-Q4_0.gguf                  364.5 MB
d1-omni-600M.Q4_K_M.gguf              235.3 MB   ← 决策模型
```

**设备端** `/home/dietpi/.cache/cactus-needle/v2/2.0.4/`：

```
libneedle.so                          13.63 MB   （权重烘焙在内）
needle2.cact                          13.10 MB
config.json / tokenizer.model / tokenizer.vocab
```

**Mac 端** `/tmp/llm-test/`、`/tmp/needle/`：下载缓存与并行下载脚本。

---

*本报告全部数据来自实际运行。所有独占内存实验结束后服务已恢复并验证（Feishu 45–80 秒重连，0 次 OOM）。*
