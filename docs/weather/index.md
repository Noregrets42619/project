# WT32-ETH01 四色墨水屏天气显示器

这次复刻的是 [henri98/esp32-e-paper-weatherdisplay](https://github.com/henri98/esp32-e-paper-weatherdisplay)。主控换成手头的 WT32-ETH01，屏幕用 Waveshare 4.2inch e-Paper Module (G)，天气接口换成 Open-Meteo，默认显示成都市郫都区的天气。

工程仓库：[Noregrets42619/esp32-e-paper-weatherdisplay](https://github.com/Noregrets42619/esp32-e-paper-weatherdisplay)。

这个项目使用普通 ESP32 的 Wi-Fi，不涉及 P4 + C5 的 ESP-Hosted 链路。显示内容是当前天气和七天预报，刷新结束后进入深度睡眠。

## 物料与接线

| 部件 | 使用情况 |
| --- | --- |
| WT32-ETH01 | ESP32 主控，工程按 4MB Flash 配置 |
| 4.2inch e-Paper Module (G) | 400 × 300，黑白黄红四色，SPI 接口 |
| DAPLink / USB 转串口 | 通过 UART0 烧录和查看日志，逻辑电平为 3.3V |
| 电源 | 调试时采用 5V 转 3.3V，屏幕接 3V3，与开发板共地 |

WT32-ETH01 板上有 RJ45，但这次沿用原工程的 Wi-Fi。板卡 5V 和 3V3 输入二选一，不能同时作为输入供电，见 [板卡说明](https://wiki.wireless-tag.com/docs/zh/WT32-ETH01/board_features.html)。

| 屏幕信号 | WT32-ETH01 GPIO | 板上标识 |
| --- | --- | --- |
| DIN / MOSI | 14 | IO14 |
| CLK / SCK | 17 | 扩展接口 TXD / TX2 |
| CS | 4 | IO4 |
| DC | 33 | 485_EN |
| RST | 32 | CFG |
| BUSY | 35 | IO35，仅输入 |
| VCC | — | 3V3 |
| GND | — | GND |

GPIO17 是扩展接口的 TX2，不是烧录串口 TXD。烧录仍走 UART0；RST 和 DC 使用板上 CFG、485_EN 对应的引脚。

## 屏幕型号

屏幕最初按“4.2 英寸、Rev2.2、V2”来判断，后来确认完整型号是 **4.2inch e-Paper Module (G)**。G 型的驱动与普通黑白 V2 不通用，这也是前期空白屏和 BUSY 超时的主要排查节点。

当前采用 [Waveshare G 型 ESP32 示例](https://github.com/waveshareteam/e-Paper/blob/master/E-paper_Separate_Program/4in2_e-Paper_G/ESP32/EPD_4in2g.cpp) 的初始化与刷新流程。BUSY 低电平忙、高电平就绪；像素采用 2 位编码，整帧 30000 字节。

组件目录仍叫 `epd4in2b`，实际编译的驱动是 `epd4in2g.c`。SPI 保持诊断时已成功显示的 100kHz。

## 中文四色界面

![中文四色天气界面示例](../img/weather/weather-ui-preview.png){ width="600" }

这是绘图代码生成的像素预览，使用示例数据，不是实机照片。界面显示中文地点、天气、星期和风向；黄色用于太阳和月亮，红色用于最高温，降水概率达到 50% 时提示栏变红。

## 当前确认的结果

| 项目 | 结果 |
| --- | --- |
| ESP-IDF 5.5.4 编译 | 通过 |
| Wi-Fi 连接和获取 IP | 串口日志已确认 |
| NTP 校时 | 串口日志已确认 |
| Open-Meteo HTTPS 请求 | 串口日志已确认当前天气与七天预报返回 |
| G 型四色诊断图 | 实机已经显示正常 |
| 中文四色天气界面 | 已编译、已生成像素预览，实机效果待烧录确认 |
| RJ45 与 OTA | 有线网络未接入，OTA 未实机验证 |

## 文档内容

1. [复刻日志](porting-log.md)：IDF 适配、接口替换、屏幕驱动排查与中文四色界面。
2. [使用与天气 API](usage-api.md)：编译烧录、Wi-Fi 密码、地点配置和 Open-Meteo 字段。
3. [报错与项目状态](issues-and-status.md)：头文件红线、空白屏、BUSY 超时与已确认结果。
