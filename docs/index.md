# Ecoli ESP32_Host_MIDI Piano 复刻笔记

![](https://cdn.jsdelivr.net/gh/Noregrets42619/blog_images/Ecoli-big.png){:height="20%" width="20%"}

这里记录的是我把 `ESP32_Host_MIDI` 的 piano 固件复刻并迁移到 WT99P4C5-S1 开发板上的过程。板子由 ESP32-P4 做主控，ESP32-C5 作为 hosted slave 提供 WiFi 和蓝牙能力；主工程使用 ESP-IDF，并把 Arduino 作为 IDF component 引入。

现在的主题是：如何尽量复用上游 Arduino MIDI 工程，在 P4+C5 硬件上做出一个可用的 MIDI Piano 固件。

## 当前目标

- 复用 `ESP32_Host_MIDI` 的 MIDI 处理和 piano 思路。
- 在 ESP-IDF + Arduino component 混合工程中完成构建。
- 使用 USB 2.0 Host 接入 class-compliant USB MIDI 设备。
- 使用 SPI 屏显示 piano UI 和按键状态。
- 使用 ES8311 + I2S 输出合成音频。
- 通过 C5 hosted WiFi 支持 AppleMIDI / RTP-MIDI。
- 通过 C5 hosted Bluetooth 支持 BLE MIDI peripheral。
- 保持 WiFi MIDI 和 BLE MIDI 可以独立开关，方便排查内存、DMA 和稳定性问题。

## 推荐阅读顺序

1. [复刻步骤](replication.md)：从原始工程到可用 MIDI Piano 固件的主流程。
2. [配置与依赖](configuration.md)：组件目录、clone 命令、`menuconfig` 关键项。
3. [移植日志](test.md)：按时间线整理的完整迁移记录。
4. [调试记录](test1.md)：USB、屏幕、音频、WiFi、BLE 关键问题和处理方式。
5. [Git 常用操作](git.md)：项目仓库和静态网页仓库的提交、构建、发布流程。

## 项目最终链路

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
