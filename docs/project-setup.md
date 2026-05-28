# 工程准备与接入记录

这一页记录项目开始阶段的工程审查、目标调整、P4C5 例程整理，以及 `ESP32_Host_MIDI` 接入 ESP-IDF 构建的过程。

## 原始工程审查

`ESP32_Host_MIDI` 原始工程不是普通 ESP-IDF 工程，而是 Arduino 库文件工程。目录中主要包含 `examples`、`src` 和说明文档。

旧工程虽然对 ESP32-P4 有支持，但主要原生功能集中在 ESP32-S3 系列芯片上，P4 部分更偏二合一 MIDI 中转器。`examples` 中的 piano 项目主要基于 `T-Display S3` 开发板和 Arduino 环境，功能被拆成多个示例，例如蓝牙 + USB、ESP-NOW + USB、屏幕显示等，并没有一个同时集成 USB、WiFi、BLE 和屏幕显示的完整项目。

## 复刻目标

基于 P4C5 开发板的外设条件，复刻目标最终确定为：以 ESP-IDF 作为主工程环境，将 USB、WiFi、BLE、屏幕显示和音频输出集成到同一个项目中，并复用 `ESP32_Host_MIDI` 的 piano 相关逻辑，实现功能集合与拓展。

P4 侧负责主应用逻辑、USB MIDI、SPI 屏显示和 ES8311 音频。C5 侧通过 ESP-Hosted 提供 WiFi 和蓝牙能力。

## C5 hosted slave 烧录

参考 ESP-IDF 官方示例 `esp32_p4_function_ev_board`，下载 `esp-hosted-mcu` 例程，并进入 `esp-hosted-mcu\slave` 目录进行编译。

C5 侧通过 USB-TTL 烧录 `hosted slave` 固件。P4 侧 `sdkconfig` 也同步进行配置，后续主要工作集中在 P4 工程本身：USB MIDI、SPI 屏、ES8311 音频、WiFi AppleMIDI 和 BLE MIDI。

## P4 工程整理

下载 ESP32-P4-C5 例程后，先剔除多余的 SPIFFS 等文件，删除不需要的组件，保留 BSP 板级支持文件，用于后续音频开发。

整理后的 P4 工程保留 ESP-IDF 主工程结构，并将 MIDI 相关代码放入 `components` 体系中，让它们作为 IDF 组件参与构建。

```text
D:\ESP_PROJECT\WT99P4C5
├── main
├── components
├── managed_components
├── sdkconfig
└── partitions.csv
```

## 组件接入

Arduino component、Host-MIDI 和显示库都曾放入组件目录。实际使用过的命令如下：

```powershell
cd D:\ESP_PROJECT\WT99P4C5\components
git clone --recursive https://github.com/espressif/arduino-esp32.git arduino-esp32
git clone --depth 1 --branch v5.2.0 https://github.com/sauloverissimo/ESP32_Host_MIDI.git ESP32_Host_MIDI
git clone --depth 1 https://github.com/lovyan03/LovyanGFX.git LovyanGFX
```

后续显示层最终没有继续依赖 LovyanGFX 做物理 SPI 驱动，`components` 中的 LovyanGFX 库也被删除。

## Host-MIDI 的 IDF 构建入口

`ESP32_Host_MIDI` 本身是 Arduino 风格，直接放入 IDF 后不能作为组件编译，因此补了 CMake 构建入口。

第一阶段只让最小功能参与编译：MIDI handler、USB Host MIDI 和 UART MIDI。BLE、WiFi 没有在一开始全部加入程序。

`main` 随后被改成最小 USB MIDI 验证入口：初始化 NVS、初始化 Arduino runtime、注册 USB MIDI transport、启动 MIDI handler，并在日志中打印 NoteOn / NoteOff。

## 早期编译问题

第一次在 VSCode 中 Reconfigure 时，工程报了 `usb/usb_host.h` 找不到。问题来自 USB 组件依赖作用域写窄：`USBConnection.h` 公开包含 `usb/usb_host.h`，所以 `usb` 必须作为公共依赖传递。CMake 依赖改成公共依赖后，这个问题解决。

接着又遇到 Brookesia 的 `Unknown memory malloc`。这个错误来自旧 UI 框架继续被工程拉进来编译，而当前主应用已经不再使用 Brookesia。旧依赖从 main manifest 中移除后，工程主线回到 MIDI。

后来 Component Manager 还在下载 `lvgl/lvgl`，并且下载下来的 `component.zip` 不是合法 zip。判断最终项目不需要 LVGL 后，BSP 切成无 LVGL 图形库模式，并移除 `lvgl/lvgl` 和 `esp_lvgl_port` 等旧显示栈依赖。

## VSCode 插件构建

当时的 IDF 环境由 VSCode ESP-IDF 插件配置，不是普通 PowerShell 直接可用的环境。后续 Reconfigure、Build、Flash 和 Monitor 都通过 VSCode 插件执行，避免普通终端中 PATH、Python venv、IDF_TOOLS_PATH 不一致导致 `idf.py` 运行异常。

## BT 配置记录

BLE 阶段，在 VSCode 的 ESP-IDF 配置界面中打开 P4 侧 BT host 和 hosted VHCI。关键配置包括：

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

配置完成后，日志中出现：

```text
Host BT Support: Enabled
BT Transport Type: VHCI
```

这两行是 P4 到 C5 蓝牙 HCI 通道已经启动的确认标志。
