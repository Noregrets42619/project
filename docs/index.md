# Ecoli ESP32_Host_MIDI Piano 移植日志

![](https://cdn.jsdelivr.net/gh/Noregrets42619/blog_images/Ecoli-big.png){:height="20%" width="20%"}

本站记录 `ESP32_Host_MIDI` piano 固件移植到 WT99P4C5-S1 开发板的过程。原始工程是 Arduino 库工程，piano 示例主要面向 T-Display S3 等 Arduino 环境；本项目在 ESP32-P4 + ESP32-C5 的硬件环境中复刻其 piano 思路，并整理成适合当前开发板的 ESP-IDF 工程。

开发板由 ESP32-P4 作为主控，ESP32-C5 通过 ESP-Hosted 提供 WiFi 和蓝牙能力。P4 侧使用 ESP-IDF，并引入 Arduino component，使上游 Arduino 工程中的 MIDI 逻辑尽量保持原结构。移植过程围绕 USB MIDI、SPI 屏、ES8311 音频、AppleMIDI 和 BLE MIDI 展开。

## 项目最终状态

- USB Host 能接入 MIDI 键盘，并收到稳定的 NoteOn / NoteOff。
- SPI 屏能显示 piano UI，按下 MIDI 键时能看到对应键位变化。
- ES8311 音频链路能发声，频率正确。
- C5 hosted WiFi 能连接热点。
- P4 能作为 AppleMIDI / RTP-MIDI 设备被手机或电脑发现。
- USB MIDI 能转发到 RTP-MIDI，手机端能收到消息并演奏。
- C5 hosted BLE 通道能启动，BLE MIDI peripheral 已接入工程。
- WiFi MIDI 和 BLE MIDI 均做成独立开关，便于按资源情况切换。

## 站点内容

1. [工程准备与接入](project-setup.md)：原始工程审查、P4C5 工程整理、C5 hosted slave 烧录和 Host-MIDI 接入记录。
2. [USB MIDI 与 SPI 显示](usb-display.md)：从 USB MIDI 验证推进到 ST7789 SPI 屏显示的记录。
3. [音频、WiFi 与 BLE](audio-network-ble.md)：ES8311、AppleMIDI 和 BLE MIDI 的接入记录。
4. [报错与处理](issues-and-fixes.md)：编译、烧录和调试中遇到的关键报错与处理结果。
5. [项目状态](project-status.md)：最终确认过的功能状态。
6. [Git 常用操作](git-commands.md)：MkDocs 站点和静态网页仓库的提交记录方式。

## 最终链路

```text
USB MIDI 键盘
    -> USB Host MIDI
    -> MIDI handler
    -> piano 状态
    -> SPI 屏显示
    -> ES8311 音频合成
    -> RTP-MIDI / BLE-MIDI 转发

手机或电脑 AppleMIDI
    -> C5 hosted WiFi
    -> RTP-MIDI
    -> MIDI handler
    -> 屏幕显示和音频输出

手机或电脑 BLE MIDI
    -> C5 Bluetooth controller
    -> ESP-Hosted VHCI
    -> P4 NimBLE host
    -> BLE MIDI transport
    -> MIDI handler
```

## Thanks

Website built by Ecoli using MkDocs. For full documentation, please visit [mkdocs.org](https://www.mkdocs.org).

Thanks for the reference of [dixi's BLOG](https://dixilog.github.io/).

![](https://cdn.jsdelivr.net/gh/Noregrets42619/blog_images/Picture%20(9).jpg){:height="20%" width="20%"}
![](https://cdn.jsdelivr.net/gh/Noregrets42619/blog_images/Picture%20(4).jpg){:height="20%" width="20%"}
