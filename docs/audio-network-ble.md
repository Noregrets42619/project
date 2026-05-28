# 音频、WiFi 与 BLE 移植记录

这一页记录 SPI 屏幕跑通后，继续接入 ES8311 音频、AppleMIDI 和 BLE MIDI 的过程。

## ES8311 音频接入

屏幕跑通后，开始处理音频。板级支持文件中已有 I2S 和 ES8311 的初始化函数，Host-MIDI 示例中也有 `SynthEngine.h`。因此音频部分沿用 Host-MIDI 的合成思路，将输出端换成 P4C5 板子上的 ES8311。

新增的音频合成层负责根据 MIDI NoteOn / NoteOff 管理复音、生成 PCM 数据，并把音频写到 ES8311。主应用中的 MIDI 事件同时进入显示层和音频层。

这里遇到一个硬件冲突：屏幕 SPI 一开始用了 GPIO7/8，而 BSP 中 ES8311 的 I2C 也在 GPIO7/8。为了保留已经调通的屏幕链路，先加了冲突保护；后续屏幕 SPI 挪到其他引脚后，音频链路能够正常初始化。

## 声音小和音量处理

音频第一次能出声后，频率正确，但声音很小。最后通过调整音量增强参数增大声音增益。重新烧录后声音恢复，并且比最开始更大。

后续又出现低频偏小、高频偏大的听感问题。处理方式不是继续单纯拉大总音量，而是加入按音高变化的增益补偿：低音区补一点，高音区压一点。这样听感比单纯提高主增益更自然。

## WiFi 和 AppleMIDI 接入

音频之后开始接 WiFi。P4 的 WiFi 由 C5 hosted slave 提供，前期已成功烧录 C5 例程，并使用 ESP-IDF 官方可用参考代码验证。

`ESP32_Host_MIDI` 的 RTP-MIDI 示例依赖 Arduino 的 `WiFiUDP`、`ESPmDNS` 和 AppleMIDI 接口，直接重写成纯 IDF UDP 会牵扯更多适配。因此 AppleMIDI 和 MIDI 库被补上 IDF 组件构建，RTP-MIDI 作为新的 MIDI transport 接入现有 `midiHandler`，同时新增 WiFi 封装层负责连接热点、启动 mDNS 和 RTP-MIDI。

这一步编译时遇到 AppleMIDI 的位判断错误：库里把 `(flags & FLAG)` 写成 `== 1`，在当前编译器和 `-Werror` 下会被判成永远 false。相关判断改成 `!= 0` 后，AppleMIDI 编译通过。

## Hosted WiFi 内存问题

WiFi 第一次运行时，`esp_hosted` 创建 SDIO mempool 时出现 `no mem`，说明 DMA 资源分配不足。处理方式是先调整 hosted SDIO 队列和不必要的 hosted 功能，压低内部 DMA 内存占用。

调完后 P4 能连上热点，MIDI 软件也能搜索到设备。

当时弹琴仍然没有传到手机端。后续确认 RTP-MIDI transport 只是作为输入接进本地 synth，USB 键盘输入不会自动发到 AppleMIDI。也就是说设备能被发现，但 USB MIDI 没有桥接到网络端。

USB transport 随后从自动队列中拆出，改为手动处理：USB 收到 MIDI 后，一份喂给本地显示和音频，一份转发给 RTP-MIDI。这样手机端能够收到 MIDI 消息并演奏。

## WiFi 与 BLE 功能开关

WiFi AppleMIDI 跑通后，日志中没有明显 mempool 或 SDIO DMA 失败，WiFi 单独运行已经可用。

为了防止 DMA 后续溢出，程序设置了 WiFi、BLE 开启标志位。如遇到 DMA 可分配资源过小，可以通过修改标志位决定开关状态。WiFi MIDI 和 BLE MIDI 都做成独立开关，用于按需要选择单独 WiFi、单独 BLE，或两者一起运行。

## P4 + C5 Hosted BLE 开启

BLE 不能按普通 ESP32 本地蓝牙处理，因为 P4 本身没有蓝牙控制器。配置路线是打开 P4 侧 Bluetooth Host、关闭本地 Controller，并启用 NimBLE + Hosted VHCI。

配置完成后，启动日志中出现：

```text
Host BT Support: Enabled
BT Transport Type: VHCI
```

看到这两行后，可以确认 P4 到 C5 的蓝牙 HCI 通道已经启动。

## BLE MIDI peripheral 接入

最后将 `ESP32_Host_MIDI` 中现有的 BLE transport 接进工程。这个过程没有额外下载 Arduino BLE 库，因为 Arduino-ESP32 component 中已经带有 BLE 相关实现。

BLE 源文件在 BT/NimBLE 开启时参与构建，并补上 BT 依赖，避免 BLE 头文件间接包含 NimBLE 头时找不到依赖。同时新增 BLE MIDI 封装层，用于初始化 BLE MIDI peripheral、维护连接状态，并把 BLE MIDI 数据送进统一的 MIDI handler。

BLE 接完后，USB MIDI 输入也能同时转发到 BLE MIDI，BLE 端发来的 MIDI 也能进入本地显示和音频链路。到这里，USB、显示、音频、WiFi AppleMIDI 和 BLE MIDI 都完成收尾。
