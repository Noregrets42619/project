# 配置与依赖

这一页集中记录复刻工程需要的组件目录、clone 命令、CMake 结构和 `menuconfig` 关键配置。

## 推荐目录结构

```text
D:\ESP_PROJECT\WT99P4C5
├── CMakeLists.txt
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml
│   └── main.cpp
├── components/
│   ├── ESP32_Host_MIDI/
│   ├── LovyanGFX/
│   ├── arduino-esp32/
│   ├── piano_app/
│   └── wt99p4c5_s1_board/
├── managed_components/
├── sdkconfig
└── partitions.csv
```

如果 Arduino 已经通过 Component Manager 出现在 `managed_components/espressif__arduino-esp32`，可以不用手动 clone 到 `components`。如果你想固定源码并方便本地修改，可以按下面方式放到 `components`。

## clone 组件

```powershell
cd D:\ESP_PROJECT\WT99P4C5\components

# Arduino as ESP-IDF component
git clone --recursive https://github.com/espressif/arduino-esp32.git arduino-esp32

# MIDI 核心库
git clone --depth 1 --branch v5.2.0 https://github.com/sauloverissimo/ESP32_Host_MIDI.git ESP32_Host_MIDI

# SPI 屏显示库
git clone --depth 1 https://github.com/lovyan03/LovyanGFX.git LovyanGFX
```

如果网络环境不稳定，可以先在浏览器或 GitHub Desktop 下载，再把目录复制到 `components`。

## CMake 依赖原则

`ESP32_Host_MIDI` 需要作为 IDF component 注册。核心原则：

- `USBConnection.h` 公开包含 `usb/usb_host.h`，所以 `usb` 要作为公共依赖传递。
- BLE 打开时，`BLEConnection.cpp` 才参与编译。
- BLE 头文件间接包含 NimBLE 头时，需要补 `bt` 依赖。
- 不要一开始把 OSC、ESP-NOW、BLE、RTP-MIDI 全部编进去，先从 USB MIDI 主线开始。

示意结构：

```cmake
idf_component_register(
    SRCS
        "src/MIDIHandler.cpp"
        "src/USBConnection.cpp"
        "src/UARTConnection.cpp"
    INCLUDE_DIRS "src"
    REQUIRES arduino usb
)

if(CONFIG_BT_ENABLED AND CONFIG_BT_NIMBLE_ENABLED)
    target_sources(${COMPONENT_LIB} PRIVATE "src/BLEConnection.cpp")
    idf_component_optional_requires(PUBLIC bt)
endif()
```

实际文件以工程中的最终 CMake 为准，这里主要记录依赖方向。

## VSCode 编译流程

推荐使用 ESP-IDF 插件提供的环境：

```text
ESP-IDF: Full Clean Project
ESP-IDF: Reconfigure Project
ESP-IDF: Build Project
ESP-IDF: Flash Project
ESP-IDF: Monitor Device
```

如果一定使用命令行，建议从 VSCode 打开：

```text
ESP-IDF: Open ESP-IDF Terminal
```

然后执行：

```powershell
idf.py reconfigure
idf.py build
idf.py flash monitor
```

不要在没有加载 ESP-IDF 环境的普通 PowerShell 里直接运行 `idf.py`。

## P4 侧 BLE menuconfig

P4 侧不是开本地蓝牙控制器，而是开 BT Host 栈，再通过 C5 hosted VHCI 使用蓝牙控制器。

关键配置：

```ini
CONFIG_BT_ENABLED=y
CONFIG_BT_CONTROLLER_DISABLED=y
CONFIG_BT_BLUEDROID_ENABLED=n
CONFIG_BT_NIMBLE_ENABLED=y
CONFIG_BT_NIMBLE_ROLE_PERIPHERAL=y
CONFIG_BT_NIMBLE_TRANSPORT_UART=n
CONFIG_ESP_HOSTED_ENABLE_BT_NIMBLE=y
CONFIG_ESP_HOSTED_NIMBLE_HCI_VHCI=y
```

VSCode 菜单大致路径：

```text
Component config
  -> Bluetooth
     -> Bluetooth: Enabled
     -> Controller: Disabled
     -> NimBLE: Enabled
     -> NimBLE Options
        -> Peripheral Role: Enabled
        -> Host-controller Transport -> UART Transport: Disabled

Component config
  -> ESP-Hosted
     -> Bluetooth Support
        -> Enable Hosted Nimble Bluetooth support: Enabled
        -> BT Nimble HCI Type: VHCI
```

启动后看日志：

```text
Host BT Support: Enabled
BT Transport Type: VHCI
```

## C5 hosted slave 配置

如果 C5 只烧了 WiFi slave，BLE 可能无法工作。C5 从机侧需要确认开启 BT sharing：

```ini
CONFIG_BT_ENABLED=y
CONFIG_BT_CONTROLLER_ONLY=y
CONFIG_BT_LE_HCI_INTERFACE_USE_RAM=y
CONFIG_ESP_HOSTED_CP_BT=y
```

菜单大致路径：

```text
Example Configuration
  -> Enable BT sharing via hosted: Enabled

Component config
  -> Bluetooth
     -> Controller only: Enabled
```

## 应用层开关

建议把功能做成独立开关：

```c
#define PIANO_APP_ENABLE_DISPLAY 1
#define PIANO_APP_ENABLE_AUDIO   1
#define PIANO_APP_ENABLE_WIFI    1
#define PIANO_APP_ENABLE_BLE     1
```

如果遇到资源压力，先关 WiFi 或 BLE 单独验证：

```c
#define PIANO_APP_ENABLE_WIFI 0
#define PIANO_APP_ENABLE_BLE  1
```

## 常用设备名和端口

```c
#define PIANO_RTP_MIDI_DEVICE_NAME "WT99P4 MIDI Piano"
#define PIANO_RTP_MIDI_PORT        5004
#define PIANO_BLE_MIDI_DEVICE_NAME "WT99P4 BLE MIDI"
```

设备名保持稳定，手机或电脑端 MIDI 软件会更容易识别和重新连接。
