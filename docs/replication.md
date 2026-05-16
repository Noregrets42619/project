# ESP32_Host_MIDI 复刻步骤

这页按实际工程推进顺序整理，从拿到 WT99P4C5-S1 原始工程，到复刻出可用 MIDI Piano 固件。

## 0. 前置条件

- P4 主工程已经能使用 ESP-IDF 构建。
- Arduino component 已经能被 IDF 工程引用。
- C5 侧已经烧录 hosted slave 固件。
- USB MIDI 设备是 class-compliant MIDI 设备。
- SPI 屏、ES8311、功放使能脚和 P4 引脚连接已确认。

## 1. 固定工程目标

原工程能力很多，但复刻阶段只保留 MIDI Piano 主线：

```text
ESP-IDF main project
    -> Arduino as IDF component
    -> ESP32_Host_MIDI
    -> USB MIDI input
    -> piano state
    -> SPI display
    -> ES8311 audio
    -> WiFi AppleMIDI
    -> BLE MIDI
```

先不要把所有旧 demo 都带着跑。Brookesia、LVGL、摄像头示例等内容可以先从构建主线移开。

## 2. 引入上游库

推荐目录结构：

```text
components/
├── arduino-esp32/          # Arduino as IDF component
├── ESP32_Host_MIDI/        # 上游 MIDI 库
├── LovyanGFX/              # SPI 屏显示库
├── piano_app/              # 当前工程自己的薄封装
└── wt99p4c5_s1_board/      # 板级支持
```

如果使用手动 clone：

```powershell
cd D:\ESP_PROJECT\WT99P4C5\components

git clone --recursive https://github.com/espressif/arduino-esp32.git arduino-esp32
git clone --depth 1 --branch v5.2.0 https://github.com/sauloverissimo/ESP32_Host_MIDI.git ESP32_Host_MIDI
git clone --depth 1 https://github.com/lovyan03/LovyanGFX.git LovyanGFX
```

如果工程已经通过 IDF Component Manager 生成 `managed_components/espressif__arduino-esp32`，也可以继续使用托管组件，不必重复 clone。

## 3. 先做最小 USB MIDI

第一阶段只做：

- 初始化 NVS。
- 初始化 Arduino runtime。
- 初始化 USB MIDI Host。
- 注册 `USBConnection`。
- 启动 `midiHandler`。
- 打印 NoteOn / NoteOff。

目标是看到类似日志：

```text
USB MIDI host initialized
NoteOn note=C4 ch=1 velocity=100
NoteOff note=C4 ch=1 velocity=64
```

这一步跑通后，说明 P4 USB Host 和 `ESP32_Host_MIDI` 核心路径成立。

## 4. 做 piano state

不要一开始就画复杂 UI。先维护一个 128 音符状态表：

```cpp
struct PianoNoteState {
    bool active;
    uint8_t velocity;
    uint8_t channel;
};

static PianoNoteState notes[128];
```

NoteOn 设置 active，NoteOff 清掉 active。显示层和音频层都从这个状态表读取。

## 5. 接 SPI 屏

屏幕层只做三件事：

- 初始化 SPI panel。
- 控制 BLK 背光。
- 根据 piano state 画键盘。

默认引脚可以集中放在 `piano_config.h`：

```cpp
#define PIANO_TFT_SCLK 7
#define PIANO_TFT_MOSI 8
#define PIANO_TFT_CS   6
#define PIANO_TFT_DC   4
#define PIANO_TFT_RST  5
#define PIANO_TFT_BL   38
```

屏幕黑屏、花屏时优先调背光极性、偏移、旋转、镜像和颜色顺序。

## 6. 接 ES8311 音频

音频层不要和 MIDI transport 耦合。主应用只把 note on/off 传给 synth：

```cpp
piano_synth_note_on(note, velocity);
piano_synth_note_off(note);
```

当前硬件上功放使能脚已确认：

```cpp
#define PIANO_POWER_AMP_ENABLE_GPIO 53
```

如果无声，先确认 GPIO53，再确认 ES8311、I2S 和合成器音量。

## 7. 接 WiFi AppleMIDI

WiFi 由 C5 hosted slave 提供，P4 侧尽量继续走 Arduino `WiFi.h`：

```cpp
WiFi.begin(PIANO_WIFI_SSID, PIANO_WIFI_PASSWORD);
```

RTP-MIDI 接入后，要确认两条路径都通：

- RTP-MIDI -> 本地显示/音频。
- USB MIDI -> RTP-MIDI 转发。

只被发现但不能演奏时，重点看 peer 数量：

```text
RTP-MIDI peers=1
```

## 8. 接 BLE MIDI

P4 没有本地蓝牙控制器，BLE 走 C5 hosted VHCI。确认日志：

```text
Host BT Support: Enabled
BT Transport Type: VHCI
```

然后接入 BLE transport：

- 编译 `BLEConnection.cpp`。
- 添加 `bt` 依赖。
- 初始化 BLE MIDI peripheral。
- 设备名例如 `WT99P4 BLE MIDI`。
- USB 输入可以同时转发到 BLE MIDI。

## 9. 做 WiFi / BLE 开关

资源紧张时，最稳的是把 WiFi 和 BLE 做成独立开关：

```c
#define PIANO_APP_ENABLE_WIFI 1
#define PIANO_APP_ENABLE_BLE  1
```

如果同时开不稳定，先单独验证：

```c
#define PIANO_APP_ENABLE_WIFI 0
#define PIANO_APP_ENABLE_BLE  1
```

## 10. 最终验收

验收顺序：

1. USB 键盘按键有干净的 NoteOn / NoteOff。
2. 屏幕能显示按键变化。
3. ES8311 能发声，GPIO53 功放使能正常。
4. AppleMIDI 软件能发现设备并建立 session。
5. USB 键盘能通过 RTP-MIDI 发送到手机或电脑。
6. 手机或电脑能通过 BLE MIDI 连接设备。
7. BLE 输入也能进入本地显示和音频链路。
