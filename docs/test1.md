# 调试记录与问题索引

这一页从原始对话记录中提取可复用的调试经验，不再保留聊天全文。它适合在遇到编译、USB、屏幕、音频、WiFi 或 BLE 问题时快速查阅。

## 编译阶段

### `usb/usb_host.h` 找不到

现象：

```text
无法打开 源 文件 "usb/usb_host.h" (dependency of "USBConnection.h")
```

处理：

- `USBConnection.h` 是公开头文件，里面包含了 `usb/usb_host.h`。
- `ESP32_Host_MIDI` 组件的 CMake 中，`usb` 需要放到公共依赖，而不是私有依赖。
- 修改后重新 `Reconfigure Project` 和 `Build Project`。

### Brookesia 报 `Unknown memory malloc`

现象：

```text
esp_brookesia_conf_kconfig.h:189: error: #error "Unknown memory malloc"
```

处理：

- 当前 MIDI 固件不再使用 Brookesia 手机 UI。
- 从 `main/idf_component.yml` 移除旧 Brookesia 依赖。
- 如果缓存还在，执行 `ESP-IDF: Full Clean Project` 后重新配置。

### Component Manager 下载 LVGL 坏包

现象：

```text
component.zip is not a zip file
```

处理：

- 当前最小 MIDI 验证不需要 LVGL。
- BSP 先切到无图形库模式。
- 移除 `lvgl/lvgl` 和 `esp_lvgl_port` 依赖。
- 如仍然下载旧依赖，清理 `dependencies.lock` 和 build cache。

## USB MIDI

### `HUB: Root port reset failed`

现象：

```text
E (...) HUB: Root port reset failed
```

判断：

- 这是 USB Host 初始化阶段常见告警。
- 如果后续能枚举设备并收到 NoteOn / NoteOff，可以先接受。
- 如果无法收包，再检查 USB 供电、线材、Hub 和设备枚举日志。

### NoteOn / NoteOff 重复两次

现象：

```text
NoteOn note=F#2 ch=10 velocity=50
NoteOn note=F#2 ch=10 velocity=50
```

处理路径：

1. 避免后台任务周期性 submit 和接收回调里再次 submit 同时存在。
2. 让 USB transport 只负责收包。
3. 主应用统一把原始数据交给 MIDI handler。
4. 必要时添加短时间窗口的 USB-MIDI packet 去重。

## SPI 屏幕

### LovyanGFX `fillScreen()` panic

现象：

```text
Guru Meditation Error: Core 0 panic'ed (Load access fault)
LGFXBase::fillScreen
PianoDisplay::begin()
```

处理：

- 不在全局构造阶段创建外设对象。
- 在 `begin()` 里延迟创建 LovyanGFX 对象。
- `tft.init()` 失败时直接返回，不再调用 `fillScreen()`。
- 初始化后检查宽高，失败时关闭显示层。
- 8 针 ST7789 模块通常按 4-wire SPI 配置，`spi_3wire = false`。

### 屏幕亮但显示异常

常见调整项：

- `PIANO_TFT_OFFSET_X`
- `PIANO_TFT_OFFSET_Y`
- `PIANO_TFT_ROTATION`
- `PIANO_TFT_MIRROR_X`
- `PIANO_TFT_MIRROR_Y`
- `PIANO_TFT_INVERT`
- `PIANO_TFT_RGB_ORDER`
- `PIANO_TFT_BL_INVERT`

调屏幕时先确认背光，再确认 SPI 通信，再调偏移和颜色。

## 音频

### 无声或声音很小

排查顺序：

1. ES8311 是否初始化成功。
2. I2S 是否正常输出。
3. MIDI NoteOn 是否进入合成器。
4. 主音量和单音增益是否过低。
5. 功放使能脚是否正确。

当前板子上 NS4150B 功放使能脚确认使用：

```text
GPIO53
```

### `dma frame num is adjusted`

这是 I2S DMA 对齐提示，不等于 DMA 溢出。只要音频连续、没有爆音或断流，可以先观察。

## WiFi / AppleMIDI

### 能发现设备但弹琴没有声音

原因：

- mDNS / AppleMIDI 服务已经被发现，但没有真正建立 RTP-MIDI session。
- 或 USB MIDI 输入只进入本地显示和合成器，没有桥接到 RTP-MIDI 输出。

判断：

```text
RTP-MIDI peers=1
USB -> RTP-MIDI bridge active
```

如果一直是 `peers=0`，说明只是被发现，还没有真正连接会话。

### hosted WiFi 日志里有 `ESP_ERR_NOT_SUPPORTED`

在 P4+C5 hosted 场景下，部分 Arduino WiFi 查询接口不完全支持。只要热点连接和 RTP-MIDI 收发正常，可以先不作为阻塞项。

## BLE MIDI

### 是否需要额外下载 Arduino BLE 库

不需要。`espressif/arduino-esp32` 组件里已经带 BLE 相关实现，`ESP32_Host_MIDI` 里也有 `BLEConnection.h/.cpp`。真正要做的是打开 BT 配置，并让 BLE 源文件参与组件构建。

### Hosted BLE 是否起来

关键日志：

```text
Host BT Support: Enabled
BT Transport Type: VHCI
```

看到这两行后，说明 P4 到 C5 的 Bluetooth HCI 通道已经起来。

### WiFi 和 BLE 是否必须同时开

不必须。推荐做成独立开关：

```c
#define PIANO_APP_ENABLE_WIFI 1
#define PIANO_APP_ENABLE_BLE  1
```

如果同时开启不稳定，可以先改成：

```c
#define PIANO_APP_ENABLE_WIFI 0
#define PIANO_APP_ENABLE_BLE  1
```

或单独 WiFi：

```c
#define PIANO_APP_ENABLE_WIFI 1
#define PIANO_APP_ENABLE_BLE  0
```

## 推荐的调试顺序

1. 先让工程能 Reconfigure 和 Build。
2. 再烧录，确认 USB MIDI NoteOn / NoteOff 干净。
3. 再打开 SPI 屏，只验证显示和按键可视化。
4. 再接 ES8311 音频。
5. 再验证 C5 hosted WiFi 和 AppleMIDI。
6. 最后验证 hosted BLE MIDI。

每一步只新增一个变量。这样一旦出问题，能快速定位是构建、USB、显示、音频、WiFi 还是 BLE。
