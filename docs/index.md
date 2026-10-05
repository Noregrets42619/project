# Ecoli ESP32 项目复刻日志

![](https://cdn.jsdelivr.net/gh/Noregrets42619/blog_images/Ecoli-big.png){:height="20%" width="20%"}

这里记录自己复刻和移植的 ESP32 项目，从选板、接线、编译到烧录后的问题处理。现在有两个工程：WT99P4C5-S1 MIDI Piano，以及 WT32-ETH01 四色墨水屏天气显示器。

## WT99P4C5-S1 MIDI Piano

把 `ESP32_Host_MIDI` 的 piano 思路移植到 ESP32-P4 + ESP32-C5。P4 负责 USB Host、屏幕和 ES8311 音频，C5 通过 ESP-Hosted 提供 Wi-Fi 与蓝牙。

USB MIDI 键盘、本地 piano 显示、音频输出，以及 AppleMIDI / BLE MIDI 的接入过程都整理在原来的日志中。

- [项目概览](piano/index.md)
- [工程准备与接入](project-setup.md)
- [USB MIDI 与 SPI 显示](usb-display.md)
- [音频、WiFi 与 BLE](audio-network-ble.md)
- [使用说明](usage-guide.md)
- [报错与处理](issues-and-fixes.md)
- [项目状态](project-status.md)

## WT32-ETH01 四色墨水屏天气显示器

复刻 `esp32-e-paper-weatherdisplay`，主控换成 WT32-ETH01，天气接口改为 Open-Meteo，屏幕采用 4.2inch e-Paper Module (G)。从普通黑白 V2 驱动超时，到确认 G 型协议、显示四色诊断图，再改成中文天气界面。

Wi-Fi 联网、NTP 校时、天气获取、中文四色显示和定时休眠均已在实物上跑通。

- [项目与硬件](weather/index.md)
- [复刻日志](weather/porting-log.md)
- [使用与天气 API](weather/usage-api.md)
- [报错与项目状态](weather/issues-and-status.md)
- [天气屏工程仓库](https://github.com/Noregrets42619/esp32-e-paper-weatherdisplay)

## 站点记录

- [Ecoli 站点](site-structure.md)
- [Git 常用操作](git-commands.md)

## Thanks

Website built by Ecoli using MkDocs. For full documentation, please visit [mkdocs.org](https://www.mkdocs.org).

Thanks for the reference of [dixi's BLOG](https://dixilog.github.io/).

![](https://cdn.jsdelivr.net/gh/Noregrets42619/blog_images/Picture%20(9).jpg){:height="20%" width="20%"}
![](https://cdn.jsdelivr.net/gh/Noregrets42619/blog_images/Picture%20(4).jpg){:height="20%" width="20%"}
