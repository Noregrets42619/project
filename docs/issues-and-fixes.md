# 报错与处理记录

这一页只记录项目中实际遇到过的错误和关键转折点。每一条都来自编译、烧录或调试时碰到的现象。

## `usb/usb_host.h` 找不到

最早接入 `ESP32_Host_MIDI` 时，VSCode Reconfigure 报过 `usb/usb_host.h` 找不到。后续确认是组件依赖作用域写窄：`USBConnection.h` 公开包含 `usb/usb_host.h`，所以 `usb` 依赖要作为公共依赖传递。

CMake 依赖改成公共依赖后，这个错误消失。

## Brookesia 的 `Unknown memory malloc`

旧工程还在拉 Brookesia，编译时报过：

```text
Unknown memory malloc
```

当前项目已经不再使用 Brookesia 手机 UI，所以旧 Brookesia 依赖被移出主工程。

## LVGL 下载坏包

后来 Component Manager 下载 `lvgl/lvgl` 时出现过：

```text
component.zip is not a zip file
```

判断最终项目不需要 LVGL 后，BSP 切成无 LVGL 图形库模式，旧 LVGL 依赖也从构建中清理。

## USB Host 初始化提示

USB MIDI 验证阶段，日志里出现过：

```text
HUB: Root port reset failed
```

后面 MIDI 设备仍然能枚举，NoteOn / NoteOff 也能收到，因此这个提示没有作为阻塞问题处理。

## MIDI 事件重复两次

USB MIDI 最开始会重复打印：

```text
NoteOn note=F#2 ch=10 velocity=50
NoteOn note=F#2 ch=10 velocity=50
```

处理方式是加入短窗口 USB-MIDI packet 去重保护。该逻辑只丢掉极短时间内完全相同的 USB-MIDI 4 字节包，并忽略 cable number。处理完成后，USB 输入变成干净事件源。

## LovyanGFX `fillScreen()` panic

第一次做屏幕时，烧录后 USB MIDI 初始化成功，但紧接着 panic。栈里反复出现：

```text
LGFXBase::fillScreen
PianoDisplay::begin
```

当时尝试过延迟创建 LovyanGFX 对象、初始化失败提前返回、避免继续 `fillScreen()` 等处理，但结果仍然继续崩。

## `LGFX_Sprite::createSprite()` 崩溃

改成 IDF `esp_lcd` 后，一开始还保留 LovyanGFX 的 `LGFX_Sprite` 当画布。屏幕能亮，但程序又在 `createSprite()` 附近崩溃。

这个现象说明物理 SPI 已经打通，真正不稳定的是 LovyanGFX 相关对象。后续彻底移除 LovyanGFX sprite，改成本地 framebuffer。物理推屏走 `esp_lcd_panel_draw_bitmap()`，上层继续保留 piano display 的绘制思路。

## 屏幕一圈花屏一圈白屏

显示不再崩后，屏幕出现一圈花屏、一圈白屏，而且按 MIDI 键时画面有变化。这个现象说明 SPI 链路已经通了，但窗口参数不对。

后续把横屏后的偏移轴调整过来，把原来加在 X 上的物理偏移换到 Y 上。再次烧录后，屏幕显示正常，按键显示也正常。

## 音频声音小

ES8311 第一次出声后，频率正确，但声音小。后续通过调整音量增强参数增大声音增益，重新烧录后声音恢复，并且比最开始更大。

低频偏小、高频偏大之后，又加入按音高变化的增益补偿，让低音区更明显，高音区没那么刺。

## AppleMIDI 位判断编译错误

接 AppleMIDI 时，编译器报过：

```text
bitwise comparison always evaluates to false
```

原因是 AppleMIDI 里把位标志判断写成 `== 1`。相关判断改成 `!= 0` 后，编译继续通过。

## hosted WiFi `no mem`

WiFi 第一次启动时，`esp_hosted` 创建 SDIO mempool 报过 `no mem`，说明 DMA 资源分配不足。当时没有重写 WiFi 层，而是先调 hosted SDIO 队列和相关配置，降低内部 DMA 内存占用。

后来热点能连上，AppleMIDI 设备也能被搜索到。

## AppleMIDI 能发现但不能演奏

手机能发现设备后，USB 键盘弹奏仍然没有传到手机。最后确认问题在桥接路径：USB MIDI 只进入本地显示和合成，没有转发到 RTP-MIDI。

USB 输入改成手动处理，同时喂给本地和 RTP-MIDI。这个改完后，手机端能收到 MIDI 消息并演奏。

## BLE 通道确认

BLE 阶段先看到日志：

```text
Host BT Support: Enabled
BT Transport Type: VHCI
```

这个日志出现后，继续接 BLE MIDI transport。最后 BLE MIDI peripheral 接入完成，工程收尾。
