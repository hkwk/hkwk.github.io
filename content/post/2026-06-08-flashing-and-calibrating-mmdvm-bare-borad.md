---
author: HKL
categories:
- Default
date: "2026-06-08T19:58:00+08:00"
slug: flashing-and-calibrating-mmdvm-bare-borad
draft: false
tags:
- OpenWrt
- Routing
- Networking
title: "From Bare Metal to BrandMeister: Flashing & Calibrating a Duplex MMDVM Hotspot Brare Board"
---

Building a digital hotspot from a bare MMDVM Dual Hat board is a rite of passage for many amateur radio operators. It’s not just plug-and-play; it’s a journey of taming RF physics, serial communications, and operating system quirks. 

After spending days troubleshooting timeout errors and frequency drifts, I finally managed to achieve a perfect 0.2% BER (Bit Error Rate) connection to the BrandMeister network. Here is the complete technical walkthrough of the process, including the hidden pitfalls that most tutorials skip.

## Phase 1: Bringing the Silicon to Life (Firmware Flashing)

When you receive a blank STM32-based MMDVM board, it’s completely brain-dead. You cannot just plug it in via USB and expect Pi-Star or MMDVMHost to recognize it.

### The Bootloader & UART Dance
1. **Initial Flash via UART:** You must use a USB-to-TTL serial adapter (UART) to write the initial firmware. Using **STM32CubeProgrammer**, connect the TX/RX pins. 
   * **Crucial Detail:** The baud rate *must* be set to `9600`. Anything higher on a blank chip will often result in connection timeouts or NAK errors.
2. **Enabling USB Native Support:** If you want to drop the UART adapter and use the STM32's native USB port for future use, you have to do a two-step flash:
   * First, flash the `stm32duino` bootloader.
   * Second, compile the MMDVM firmware with USB support enabled. When flashing this payload, you must set the memory address offset to `0x2000` so it doesn't overwrite the bootloader.

## Phase 2: The RF Calibration Crucible (Finding the Offset)

Even with the firmware running, the hotspot is "deaf" until you calibrate the `RXOffset`. Hardware crystals (especially the 14.7456MHz TCXOs used on these boards) have manufacturing tolerances. If the offset is wrong, your radio's DMR signals will be ignored.

### Using MMDVMCal
1. Connect the board to your PC and run `./MMDVMCal <port> 115200`.
2. Press `E` to set your target frequency (e.g., `434150000` Hz).
3. Press `b` to enter **BER Test Mode (FEC) for DMR Simplex**.
4. Hold down the PTT on your DMR radio.
5. While holding PTT, press `F` (increase) or `f` (decrease) on your keyboard to sweep the frequency in 10Hz/50Hz steps until the screen suddenly displays `DMR voice header received` and starts calculating the BER.
6. Keep tuning until the BER drops below 1% (ideally 0.0%). The difference between your radio's frequency and the frequency shown on screen is your `RXOffset`.

### 🛑 Pitfall #1: The "One-Hit Wonder" (Thermal Drift)
**The Problem:** I found my offset, put it in `MMDVM.ini`, and successfully made *one* transmission. But when I pressed PTT a second time, the hotspot ignored me (TIMEOUT). 
**The Cause:** Most budget duplex boards use a **single TCXO** shared between the TX and RX chips (ADF7021). When you transmit, the TX chip generates heat. This heat transfers to the shared TCXO, causing its frequency to temporarily drift. My offset was literally changing in real-time!
**The Solution:** Do not calibrate a cold board. Let the hotspot idle for 10-15 minutes to reach **thermal equilibrium**. Run the MMDVMCal test again while the board is warm. 
Furthermore, I noticed my cold offset was `+3050` and my hot offset was `+2600`. I set my `RXOffset` to **`+2800`** (the golden middle ground). Since DMR has an AFC (Automatic Frequency Control) tolerance of about ±500Hz, this middle value guarantees a perfect connection regardless of the board's temperature!

## Phase 3: The OS Latency Trap (Windows vs. Linux)

Because this is a single-TCXO board, if the RX is physically offset by `+2800Hz`, the TX is physically offset by the exact same amount. 

**The Mystery:** * On **Linux**, I left `TXOffset=0`, and my radio received the hotspot perfectly.
* On **Windows**, leaving `TXOffset=0` caused my radio to fail to decode the signal. It only worked when I set `TXOffset=2800`.

**The Explanation:**
Windows USB VCP (Virtual COM Port) drivers introduce much higher micro-jitter and packet latency compared to the highly optimized Linux `cdc_acm` kernel drivers. 
DMR is highly timing-dependent. On Linux, the timing is so precise that the radio's AFC can comfortably absorb the 2800Hz frequency error. On Windows, the USB latency "eats up" the radio's error-correction budget. Therefore, on Windows, you **must** configure `TXOffset` to perfectly match `RXOffset` to give the radio's decoder a fighting chance.

## Phase 4: Taming the Network (DMRGateway & CPS Setup)

Transitioning from a simplex OpenGD77 hotspot to a true Duplex MMDVM means a paradigm shift: you now have **two independent time slots**.

### MMDVMHost Configuration
In your `MMDVM.ini`, ensure duplex and both slots are enabled:
```ini
[General]
Duplex=1

[DMR]
Slot1=1
Slot2=1

```

### DMRGateway TGRewrite Magic

To cleanly route traffic from BrandMeister, you use `TGRewrite` rules in `DMRGateway.ini` to lock certain Talk Groups to specific slots as "Static Groups".

```ini
# Route TG91 (World) and TG46007 (China) permanently to Slot 2
TGRewrite=2,91,2,91,1
TGRewrite=2,46007,2,46007,1

```

### 🛑 Pitfall #2: Radio CPS "Rx Group Lists"

The biggest mistake you can make is configuring the hotspot perfectly but failing to set up your radio. Because the hotspot is transmitting static groups (like TG91) without you actively pressing PTT to request them, your radio will ignore the audio unless instructed otherwise.

1. In your radio's CPS, create a **Digital Contact** for TG91.
2. Create an **Rx Group List** and add TG91 to it.
3. In your Channel configuration, ensure the **Repeater Slot** is set correctly (e.g., Slot 2), and **MUST** attach the Rx Group List you just created.

## Conclusion

Building this hotspot was a masterclass in RF physics and serial communications. By understanding the limitations of shared TCXOs, accommodating for thermal drift, and properly configuring the gateway, a "cheap" generic dual hat can be tuned to deliver commercial-grade performance with a 0.2% BER.

See you on Air! **73 de BG7KUH.**

![MMDVM UART Connection](https://cdn.jsdelivr.net/gh/hkwk/blog-photo/2026/06/01.jpg)

![MMDVM HS Dual Hat](https://cdn.jsdelivr.net/gh/hkwk/blog-photo/2026/06/02.jpg)


## Chinese Version

对于许多业余无线电爱好者（火腿）来说，从零折腾一块空白的 MMDVM 双工扩展板（Dual Hat）绝对是必经的“受难与进阶”之路。这可不是简单的即插即用，而是一场与射频物理学、串口通信时序以及操作系统底层逻辑的博弈。

在经历了数天的超时报错（Timeout）和频偏迷阵后，我终于成功将这块双工板接入了 BrandMeister 网络，并跑出了 **0.2% 极低误码率（BER）** 的完美成绩。本文将完整复盘整个技术流程，并毫无保留地分享那些大多数教程里不会告诉你的“深坑”。

## 第一阶段：唤醒硅片（固件烧录）

当你拿到一块全新的基于 STM32 的 MMDVM 裸板时，它是没有“灵魂”的。你不能指望把它插到 USB 上就能被 Pi-Star 或 MMDVMHost 识别。

### 串口（UART）与 Bootloader 的舞蹈
1. **初次串口烧录**：必须使用 USB 转 TTL 模块（UART）连接板子上的 TX/RX 引脚来进行底层的固件写入。使用 **STM32CubeProgrammer** 软件时，**核心细节是：波特率必须且只能设为 `9600`**。如果在空白芯片上使用更高的波特率，极易导致连接超时或 NAK 报错。
2. **开启原生 USB 支持（进阶）**：如果你想摆脱 UART 模块，直接用板子上的 USB 接口通信，需要进行“两步走”烧录：
   * 第一步：先烧录 `stm32duino` 的 bootloader。
   * 第二步：编译带有 USB 支持的 MMDVM 固件。在通过串口烧录这个固件时，**必须将内存地址偏移量（Offset）设置为 `0x2000`**，否则会覆盖掉刚才辛苦刷入的 bootloader！

## 第二阶段：射频调校大考（抓取频偏 Offset）

固件跑起来后，热点依然是“聋子”。由于硬件晶振（特别是这种板子常用的 14.7456MHz TCXO 温补晶振）存在制造公差，如果找不到正确的频偏（RXOffset），芯片将直接无视你手台发出的 DMR 信号。

### MMDVMCal 实战
1. 将热点连接电脑，运行 `./MMDVMCal <串口号> 115200`。
2. 按 `E` 输入你的目标测试频率（例如 `434150000` Hz）。
3. 按 `b` 进入 **DMR 单工误码率测试模式（BER Test）**。
4. 按住你的 DMR 手台的 PTT 发射不松手。
5. 在键盘上按 `F`（增加）或 `f`（减小），以 10Hz/50Hz 的步进扫描，直到屏幕突然跳出 `DMR voice header received` 并开始计算 BER 误码率。
6. 继续微调，直到 BER 降到 1% 以下（最好是 0.0%）。此时屏幕显示的数值差就是你的 `RXOffset`。

### 🛑 避坑指南 #1：“昙花一现”的通联（温漂效应）
**问题现象**：我找到了 Offset 并填入 `MMDVM.ini`，第一次发射完美通联，但第二次按 PTT 就立刻提示 TIMEOUT（超时无反应）。
**底层真相**：市面上大部分高性价比的双工板，为了省成本，RX 和 TX 两颗 ADF7021 射频芯片**共用一颗 TCXO 晶振**。当你第一次发射时，TX 芯片产生的大量热量直接传导给了这颗晶振，导致频率在瞬间发生了几百赫兹的物理漂移！我刚才测出的参数瞬间失效了。
**终极解法**：**绝对不要在冷机状态下测频偏。** 先让热点开机空载运行 10-15 分钟，达到“热平衡状态”。
* 实测发现：我的板子冷机频偏是 `+3050`，热透了之后是 `+2600`。
* 既然 DMR 的 AFC 容错率大约是 $\pm 500\text{Hz}$，我直接取了一个**黄金中间值 `+2800`**。这样无论冷机还是热机，频偏都在手台的容错范围内，彻底解决了掉线问题！

## 第三阶段：操作系统延迟陷阱（Windows vs. Linux）

既然是单晶振双工板，RX 物理上偏了 +2800Hz，TX 必然也偏了 +2800Hz。但在接下来的测试中，我遇到了极其诡异的跨平台问题。

**灵异现象**：
* 在 **Linux** 下，我的 `TXOffset` 设为 `0`，手台依然能完美接收。
* 在 **Windows** 下，`TXOffset` 设为 `0` 手台直接变哑巴，**必须**设为 `2800` 手台才出声。

**真相大白**：
这其实是操作系统的 USB 虚拟串口（VCP）底层驱动惹的祸。
DMR 是极度依赖时隙切换时序的。Linux 内核的 `cdc_acm` 驱动优化极佳，延迟几乎为零，手台在接收时吃满了时序红利，所以其内部强大的 AFC（自动频率控制）硬生生把那 2800Hz 的物理频偏给“包容”了。
而在 Windows 下，USB 驱动自带较高的缓冲延迟（Jitter），时序上的微弱抖动消耗了手台的容错率，手台再也无力去纠正那 2800Hz 的频偏了。因此，在 Windows 下你必须老老实实地把物理频偏 `TXOffset=2800` 补齐，才能让手台正常解码。

## 第四阶段：驯服网络（DMRGateway 与写频精要）

从以前玩 OpenGD77 的单工热点，升级到真正的 MMDVM 双工热点，最大的爽点就是：**时隙 1 (Slot 1) 和时隙 2 (Slot 2) 终于可以独立且同时工作了！**

### 巧用 DMRGateway 配置静态组
在 `DMRGateway.ini` 中，通过 `TGRewrite` 规则，我们可以把常用的组（比如 TG91 国际组、TG46007 全国组）死死绑定在时隙 2 上，变成“静态组”：
```ini
# 将 TG91 和 TG46007 映射到时隙 2
TGRewrite=2,91,2,91,1
TGRewrite=2,46007,2,46007,1

```

### 🛑 避坑指南 #2：手台 CPS 的“接收组列表”

这是新手最容易踩的坑。热点配置得再完美，如果手台没设置对，你依然听不到声音。因为对于这种网络主动下发的“静态组”流量，手台默认是静音的。

1. 在写频软件（CPS）中，新建一个组呼联系人（如 `TG91`）。
2. **最关键的一步**：新建一个**接收组列表（Rx Group List）**，把 `TG91` 放进去。
3. 在你的信道设置里，确保**异频设置正确**，选对**时隙 2（Repeater Slot 2）**，并且**必须挂载**刚才建好的接收组列表。

## 结语

调校这块双工热点的过程，堪称一堂生动的射频物理与软硬件结合的大课。当你洞悉了共用晶振的热漂移规律、摸清了系统的底层时序，并利用规则驯服了数字网络后，那种听着 0.2% BER 丝滑语音的成就感，是无可比拟的。

期待空中相遇！**73 de BG7KUH**。


