# USB MIDI 与 SPI 显示移植记录

这一页记录从 USB MIDI 验证推进到 ST7789 SPI 屏显示跑通的过程。

## USB MIDI 烧录验证

编译通过后，先烧录最小固件并接入 USB MIDI 设备验证数据链路。串口能看到 NoteOn / NoteOff，说明 P4 的 USB Host 能识别 MIDI 设备。

当时日志中出现两个现象：

- 上电或者插入 MIDI 键盘后，有 `HUB: Root port reset failed`。
- 同一个 MIDI 事件会打印两次。

`Root port reset failed` 后续不影响 MIDI 收包，因此认定为 USB Host 初始化阶段的非致命提示。重复打印更影响调试，处理方式为加入一个很窄的 USB-MIDI packet 去重保护。

这个保护只丢掉极短时间内完全相同的 USB-MIDI 4 字节包，并忽略 cable number。处理完成后，USB MIDI 输入变干净，piano 层可以继续向上接。

## Piano 状态层

USB MIDI 正常后，开始搭建 piano 状态层。屏幕尺寸和 Host-MIDI 示例中的 piano display 接近，P4C5 开发板的小屏幕通过排针走 SPI 协议，8 针屏幕中的 BLK 单独使用一个 GPIO 控制。

状态层主要维护 MIDI note 的按下状态、力度和通道。后续屏幕显示和音频发声都从这套状态出发。

## LovyanGFX 显示尝试

最初尝试延续原工程，使用 Arduino LovyanGFX 库，将 piano 状态和显示层拆开：一个部分维护 128 个 MIDI note 状态，另一个部分绘制 ST7789 170x320 的 piano UI。

默认 SPI 引脚先按当时引出的 IO 设定：

```text
SCLK = 7
MOSI = 8
CS   = 6
DC   = 4
RST  = 5
BLK  = 38
```

LovyanGFX 被拷贝进 components，用于尝试复用其屏幕驱动能力。

## 显示层 panic

第一次烧录后，USB MIDI host 初始化成功，但随后发生 panic。栈中显示崩在 LovyanGFX 的 `fillScreen()`，`PianoDisplay::begin()` 访问屏幕宽高时出错，屏幕没有显示。

后续做过几次尝试：把 LovyanGFX 对象改成运行时创建，避免全局构造阶段初始化外设对象；在 `tft.init()` 失败时提前返回，不再继续 `fillScreen()` 等。

这些处理完成后仍然 panic。新的日志依旧卡在 `fillScreen()` 和 `width()`，说明问题集中在 LovyanGFX 的 panel 状态，而不是钢琴绘制逻辑。

## 改用 IDF esp_lcd 直接驱动 SPI

最终决定重写 SPI 方式。目标是上层仍按 Host-MIDI 的 PianoDisplay 绘制思路组织，但物理推屏不再走 Arduino/LovyanGFX 显示库，而是改成 IDF 原生 `esp_lcd` 驱动 ST7789。

这一步没有大改业务逻辑，只替换底层显示后端。先用 `esp_lcd` 初始化 SPI ST7789，再用 `esp_lcd_panel_draw_bitmap()` 推整帧。

最开始还保留 LovyanGFX 的 `LGFX_Sprite` 当画布，但烧录后屏幕亮起并出现花屏，程序崩在 `LGFX_Sprite::createSprite()`。这个现象说明物理 SPI 已经打通，真正不稳定的是 LovyanGFX 相关对象。

随后 `LGFX_Sprite` 也被移除，改成本地 framebuffer 画布，只保留 piano 需要的矩形、线条和简单文字绘制。至此显示层完全避开 Arduino 显示库，底层走 IDF `esp_lcd`，上层继续保留 Host-MIDI 的 piano display 思路。`components` 中的 LovyanGFX 库也被删除。

## SPI 屏幕调通

换成 `esp_lcd` 后，屏幕能亮，日志也显示 ST7789 SPI panel 初始化成功。后面的问题变成屏参问题：一圈花屏，一圈白屏，但是按 MIDI 键时画面有变化。

这个现象说明 SPI 写屏链路已经工作，问题集中在窗口偏移、横竖屏和像素格式上。后续确认 `swap_xy(true)` 横屏后，原来 Host-MIDI 示例中的物理偏移轴需要调整。之前偏移加在 X 上，写入窗口会越界，因此出现一边花、一边白。

偏移最终改成 X=0、Y=35，并将 endian、mirror、swap、RGB 顺序等屏参集中成宏。再次烧录后，屏幕正常显示，接入 MIDI 键盘时也能显示按下的键位。至此 SPI 屏幕显示移植正常。

## 阶段结果

- USB MIDI 输入正常。
- USB-MIDI 重复事件已处理。
- piano 状态层能接收按键变化。
- ST7789 SPI 屏初始化成功。
- 本地 framebuffer 能正常推屏。
- 按下 MIDI 键盘时，屏幕能显示对应键位。
