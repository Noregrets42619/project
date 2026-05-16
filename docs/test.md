# WT99P4C5 MIDI Piano 移植日志

这篇日志记录了我在 WT99P4C5-S1，也就是 ESP32-P4 + ESP32-C5 板子上，把 `ESP32_Host_MIDI` 的 piano 固件迁移成一个可用 MIDI Piano 工程的过程。

目标是尽量复用上游 Arduino 工程，减少重写量，让工程在 ESP-IDF + Arduino component 混合编译环境下稳定运行，并支持 USB MIDI、屏幕显示、音频输出、WiFi AppleMIDI 和 BLE MIDI。

## 项目背景

原始工程是一个面向 WT99P4C5-S1 的 ESP-IDF 综合示例，带有 Brookesia/LVGL、摄像头、音频、显示等能力。它适合作为板级能力参考，但不适合作为 MIDI Piano 主应用直接继续堆功能。

重新定义后的目标如下：

- 主控是 ESP32-P4。
- WiFi 和蓝牙能力由从机 ESP32-C5 通过 ESP-Hosted 提供。
- C5 已经烧录并配置好 hosted slave 固件。
- P4 侧使用 ESP-IDF，并引入 Arduino component 混合编译。
- MIDI 核心逻辑尽量复用 `ESP32_Host_MIDI` 的上游 Arduino 代码。
- 屏幕从上游示例迁移到 SPI 协议。
- USB 作为 Host 接入 MIDI 键盘。
- WiFi 支持 AppleMIDI / RTP-MIDI。
- BLE 作为 BLE MIDI peripheral 被手机或软件发现连接。
- 板载音频使用 ES8311，外接 NS4150B 功放。

## 1. 重新定位工程

第一步是确认工程本质：它是 ESP-IDF/CMake 工程，不是普通 Arduino 工程。旧工程里的 Brookesia、LVGL、摄像头示例都不是 MIDI piano 主线。

我把工程收敛成：

- ESP-IDF 作为主构建系统。
- Arduino 作为 IDF component 使用。
- `ESP32_Host_MIDI` 作为上游库接入。
- 只保留 MIDI piano 所需依赖。
- 不继续沿用旧 Brookesia 手机 UI 作为主应用。

## 2. 接入 ESP32_Host_MIDI

上游库复制到 `components/ESP32_Host_MIDI` 后，需要补 IDF 组件构建入口。第一版只纳入最小可验证能力：

- MIDI 核心 handler。
- USB Host MIDI。
- UART MIDI。

先不要把 BLE、RTP-MIDI、OSC、ESP-NOW 全部一次性编进来。这样可以先确认 `MIDIHandler + USBConnection + Arduino as IDF component` 这条主线稳定。

关键点是 USB 依赖必须作为公共依赖传递，因为 `USBConnection.h` 会公开包含 `usb/usb_host.h`。如果 `usb` 只放在私有依赖里，`main` 间接 include 时会找不到头文件。

## 3. 清理旧依赖

旧工程的 Brookesia/LVGL 依赖会干扰 MIDI 固件构建。我做了这些处理：

- 移除不再需要的 Brookesia 依赖。
- 暂停下载 `lvgl/lvgl` 和 `esp_lvgl_port`。
- 保留后续真正需要的板级能力，例如 SPI、GPIO、USB、音频、WiFi remote。
- BSP 侧用无图形库模式先跑通主线。

这一步之后，工程从综合 demo 变成更清晰的 MIDI 固件工程。

## 4. 建立最小 USB MIDI 验证入口

主程序先只做最小验证：

- 初始化 NVS。
- 初始化 Arduino runtime。
- 初始化 USB MIDI Host。
- 注册 MIDI 数据回调。
- 调用 MIDI handler 解析收到的数据。
- 在串口日志里打印 NoteOn / NoteOff。

验证通过后，P4 的 USB Host 可以识别并读取 class-compliant USB MIDI 设备。

## 5. 处理 USB MIDI 重复事件

最开始同一个音符事件会打印两次。处理思路是让 USB transport 只负责收包，由主应用统一把原始数据交给 MIDI handler，避免同一份数据被 transport 队列和手动回调重复消费。

如果仍然重复，可以加一个很窄的 USB-MIDI packet 去重保护：

- 只在极短时间窗口内去掉完全相同的 4 字节 USB-MIDI packet。
- 比较时忽略 cable number，用来兼容设备通过多个 virtual cable 发同一份数据的情况。
- 不影响正常快速连按，因为窗口非常短。

## 6. 移植 piano 显示层

屏幕部分按上游 piano display 的思路做最小迁移：

- `PianoState` 维护 128 个 MIDI note 的状态。
- `PianoDisplay` 负责绘制 ST7789 SPI 屏。
- 屏幕方向使用横屏 320x170。
- 默认绘制 25 键 piano view。
- 背光单独用 GPIO 控制。
- 所有屏幕引脚、偏移、旋转、颜色顺序集中到 `piano_config.h`。

默认引脚曾按可用 IO 先定为：

```cpp
SCLK = 7
MOSI = 8
CS   = 6
DC   = 4
RST  = 5
BLK  = 38
MISO = -1
```

调试时遇到过 `fillScreen()` panic、panel 指针为空、屏幕亮但花屏、一圈花屏、一圈白屏等问题。最终通过延迟创建 LovyanGFX 对象、检查初始化结果、修正 4-wire SPI、调整 offset/rotation/RGB 顺序解决。

## 7. 接入 ES8311 音频

音频部分参考板级支持代码里的 I2S 和 ES8311 初始化逻辑，同时参考上游 piano 示例里的 synth engine 思路。

完成内容：

- 初始化 ES8311 codec。
- 建立 I2S 输出。
- 把 MIDI NoteOn / NoteOff 映射成合成器发声和停音。
- 让显示、MIDI 解析、音频响应共用同一套 note 状态。
- 增加主音量、单音增益、低频增益、高频增益等参数。
- 确认 NS4150B 功放使能脚是 GPIO53。

调试中一度出现声音很小或无声。最终确认功放使能脚后，音频链路恢复。

## 8. 接入 WiFi Hosted 和 AppleMIDI

P4 的 WiFi 由 C5 hosted slave 提供，因此 P4 侧不重写 WiFi 底层，而是让 Arduino `WiFi.h` 走 ESP-Hosted 适配路径。

完成内容：

- 使用 Arduino WiFi 接口连接热点。
- 接入 AppleMIDI / RTP-MIDI。
- 加入 mDNS 服务发现。
- 让手机或 MIDI 软件能发现当前设备。
- 把 USB MIDI 输入转发到 RTP-MIDI。
- 让 RTP-MIDI 传入的数据进入显示和音频逻辑。

一个关键问题是：一开始设备可以被 MIDI 软件发现，但弹琴没有声音。原因是 USB MIDI 收到的数据只进入本地显示和合成器，没有真正桥接到 RTP-MIDI 输出。补上 USB -> RTP-MIDI 转发后，手机可以收到 MIDI 消息并演奏。

## 9. 分析 DMA 与运行日志

WiFi MIDI 跑通后，日志里重点看这些内容：

- hosted WiFi 是否 ready。
- RTP-MIDI 是否 ready。
- 是否出现 mempool 或 DMA 严重失败。
- I2S 是否只是 `dma frame num is adjusted` 这类对齐提示。
- RTP-MIDI peer 数量是否从 0 变成 1。

结论是：WiFi 单独运行时 DMA 内存够用。BLE MIDI 数据量很小，真正压力主要来自 BT host 栈内存、WiFi hosted 队列和音频 DMA，所以 WiFi 与 BLE 最好做成可切换，而不是强制永远并发。

## 10. 配置 Hosted BLE

P4 本身没有蓝牙控制器，所以不能按普通 ESP32 本地 BLE 配置。最终路线是：

- P4 侧开启 Bluetooth Host。
- P4 侧关闭本地 Bluetooth Controller。
- P4 侧使用 NimBLE。
- P4 侧启用 Hosted NimBLE over VHCI。
- C5 侧作为 hosted slave 提供 Bluetooth controller。

关键验证日志：

```text
Host BT Support: Enabled
BT Transport Type: VHCI
```

看到这两行后，说明 P4 到 C5 的蓝牙 HCI 通道已经起来。

## 11. 接入 BLE MIDI peripheral

Hosted BLE 配置成功后，把 BLE MIDI 接进现有 piano 应用：

- 让 `BLEConnection.cpp` 在 BT/NimBLE 开启时参与构建。
- 给 CMake 添加 `bt` 依赖，避免 BLE 头文件间接包含 NimBLE 头时找不到依赖。
- 把 BLE 初始化结果改成可返回成功或失败。
- 新增 `piano_ble` transport 封装层。
- 在主应用里初始化 BLE MIDI。
- 将 BLE MIDI transport 注册到统一 MIDI handler。
- 把 USB MIDI 输入同时转发到 BLE MIDI。
- BLE 端发送来的 MIDI 也进入现有显示和音频链路。
- BLE 断开时清理活动音符，避免挂音。

## 12. 做成 WiFi / BLE 可切换

最终配置设计：

- `PIANO_APP_ENABLE_WIFI` 控制 WiFi / AppleMIDI。
- `PIANO_APP_ENABLE_BLE` 控制 BLE MIDI。
- 两者可以尝试同时开启。
- 如果出现内存紧张、初始化失败或延迟抖动，优先改成单独 BLE 或单独 WiFi 模式。

这样后续调试不需要重写代码，只需要改配置。

## 总结

这个工程已经从原来的 WT99P4C5-S1 综合示例，迁移成一个完整的 MIDI Piano 固件。整体思路是最小化重写、最大化复用上游 Arduino MIDI 代码，同时用 IDF 组件方式把每个功能稳定接进 P4+C5 的实际硬件环境。
