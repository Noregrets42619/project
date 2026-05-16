# 功能状态与复盘

这一页汇总当前 MIDI Piano 复刻工程已经跑通的功能、仍需关注的风险点，以及后续维护建议。

## 当前功能状态

| 功能 | 状态 |
| --- | --- |
| ESP-IDF + Arduino component 混合编译 | 已跑通 |
| `ESP32_Host_MIDI` 接入 | 已完成 |
| USB MIDI Host | 已可用 |
| MIDI 事件解析 | 已可用 |
| SPI 屏 piano UI | 已可用 |
| 按键可视化 | 已可用 |
| ES8311 音频输出 | 已可用 |
| NS4150B 功放使能 | 已确认 GPIO53 |
| C5 hosted WiFi | 已可用 |
| AppleMIDI / RTP-MIDI | 已可用 |
| USB 到 RTP-MIDI 转发 | 已可用 |
| BLE Hosted VHCI | 已可用 |
| BLE MIDI peripheral | 已接入 |
| WiFi / BLE 功能切换 | 已支持 |

## 已确认的非致命现象

- `HUB: Root port reset failed`：USB Host 初始化阶段可能出现，只要后续 MIDI 设备能枚举和收包，可以先观察。
- `dma frame num is adjusted`：I2S DMA 参数对齐提示，不等于 DMA 溢出。
- `RTP-MIDI peers=0`：只是当前没有 AppleMIDI session 连接，不代表服务异常。
- hosted WiFi 场景下某些 Arduino WiFi 查询返回 `ESP_ERR_NOT_SUPPORTED`：不影响连接热点和 RTP-MIDI 使用。

## 核心复盘

这个工程最重要的路线不是“重写一个 piano 固件”，而是把上游 Arduino MIDI 工程稳稳接入 IDF 世界：

1. 先让 `MIDIHandler + USBConnection + Arduino component` 编译通过。
2. 再把 USB MIDI 输入作为干净事件源。
3. 然后逐层接入 piano 状态、SPI 显示、ES8311 音频。
4. WiFi 和 BLE 不抢在第一阶段做，等本地 MIDI 闭环稳定后再加。
5. P4 本体没有 WiFi/BT 射频，所以 hosted 通道必须单独验证。

## 后续维护建议

- 不要直接大改上游 `ESP32_Host_MIDI` 核心逻辑，尽量把板级差异放进 wrapper 和配置层。
- USB、WiFi、BLE、音频、显示要保持模块独立，不要互相硬耦合。
- 如果同时开启 WiFi 和 BLE 出现内存紧张或延迟抖动，优先采用二选一模式。
- 屏幕换型号时，优先调整 SPI host、偏移、镜像、颜色顺序和背光极性。
- 音频继续优化时，优先调整合成器增益、包络、滤波和音量，不要先动底层 I2S。
- 升级 ESP-IDF 或 Arduino-ESP32 后，要重点复查 NimBLE hosted、USB Host 和 CMake 依赖传递。
