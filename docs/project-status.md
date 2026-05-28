# 项目状态

这里记录最终确认过的项目状态。页面只保留当时确认到的结果。

## 已经跑通的功能

| 功能 | 确认结果 |
| --- | --- |
| ESP-IDF + Arduino component 混合编译 | 已经跑通 |
| `ESP32_Host_MIDI` 接入 | 已经接入 IDF 构建 |
| USB MIDI Host | 能收到键盘输入 |
| MIDI 事件解析 | NoteOn / NoteOff 正常 |
| USB 重复事件 | 已经处理 |
| SPI 屏显示 | 已经正常显示 |
| 按键可视化 | 敲击键盘时能显示键位 |
| ES8311 音频 | 能发声，频率正确 |
| 低频/高频听感 | 已加音高增益补偿 |
| C5 hosted WiFi | 能连接热点 |
| AppleMIDI / RTP-MIDI | 手机能发现并接收 MIDI |
| USB 到 RTP-MIDI 转发 | 已经打通 |
| BLE Hosted VHCI | 日志确认已启用 |
| BLE MIDI peripheral | 已经接入 |
| WiFi / BLE 功能切换 | 已经做成独立开关 |

## 接受下来的日志现象

`HUB: Root port reset failed` 在 USB Host 初始化时出现过，但 MIDI 设备后续能正常使用。

`dma frame num is adjusted` 在 I2S 初始化时出现过，它只是 DMA 参数对齐提示，不影响当前的发声结果。

`RTP-MIDI peers=0` 只表示当时还没有 AppleMIDI 会话连接。连接之后，peer 数量会变化，MIDI 消息也能传过去。

## 最后的项目链路

```text
USB MIDI 键盘
    -> USB Host MIDI
    -> 本地 piano 状态
    -> SPI 屏显示
    -> ES8311 发声
    -> RTP-MIDI / BLE-MIDI 转发

AppleMIDI
    -> C5 hosted WiFi
    -> RTP-MIDI
    -> 本地 piano 状态
    -> SPI 屏显示
    -> ES8311 发声

BLE MIDI
    -> C5 hosted Bluetooth
    -> P4 NimBLE host
    -> BLE MIDI transport
    -> 本地 piano 状态
    -> SPI 屏显示
    -> ES8311 发声
```
