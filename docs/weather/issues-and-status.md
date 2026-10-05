# 天气屏报错与项目状态

这里保留复刻过程中遇到的现象和确认到的结果。记录更新到 2026-10-05。

## 编译成功，头文件仍然标红

问题列表中的 `C/C++(1696)` 来自 VS Code IntelliSense。曾同时报出 `esp_wifi.h`、`driver/gpio.h`、`math.h`、`stdint.h` 找不到，但 IDF 构建已经成功。

后来调整 ESP32 GCC 路径，并让编辑器使用 `build/compile_commands.json` 和生成的 `sdkconfig.h`。必要时执行 `idf.py reconfigure`，再重新加载窗口或重置 IntelliSense 数据库。

删除 `build` 后，编译数据库也会消失，需要重新生成。开发阶段暂时保留构建目录。

## 网络成功，但屏幕空白

Wi-Fi 获得 IP、NTP 成功和 HTTPS 返回天气，只能说明网络链路工作。早期即使打印了 `e-Paper initialized`，屏幕也仍是空白。

当时的 `EPDIF: All OK` 使用了错误日志级别，后来改成信息级别。屏幕没有图像时，继续检查完整型号、驱动指令、BUSY 极性和接线，不能把这行日志当成显示成功。

## 黑白 V2 驱动的 BUSY 超时

旧测试日志如下：

```text
epd4in2_v2: hardware reset: BUSY GPIO35 remained HIGH for 45s
screen_diagnostic: Test failed: ESP_ERR_TIMEOUT
```

一开始按 V2 标记选了普通黑白屏驱动，后来确认实际屏幕是带 **(G)** 的四色型号。改用 G 型初始化流程后，四色诊断图实机显示成功。

当前 G 型的 BUSY 是低电平忙、高电平就绪，与旧 V2 测试相反。刷新时会等待 BUSY 的 LOW→HIGH 周期，最长等待 90 秒。

| 当前日志 | 含义 |
| --- | --- |
| `No BUSY LOW after G refresh command` | 没检测到刷新响应 |
| `remained LOW for 90s` | 屏幕在等待窗口内没有回到就绪 |
| `G full refresh BUSY cycle completed` | 已观察到刷新周期，最终还要看屏幕内容 |
| `Image transfer` | SPI 图像发送耗时 |
| `full refresh: ready HIGH after ... ms` | 屏幕 BUSY 等待耗时 |

## 休眠后测到 BUSY 为 0V，RST 约 0.6V

这组读数是在程序进入深度睡眠之后测得。后面增加单屏诊断，让 ESP32 保持唤醒，并在失败后留出 60 秒窗口，RST 持续输出高电平，再进行测量。

窗口结束后程序会明确把 RST 拉低。因此排查时要区分初始化阶段、测量窗口和停止后的状态。

烧录使用 USB 转 TTL，调试供电为 5V 转 3.3V。3.3V 供电本身并不能解释最初的超时，最终确认到的关键差异是屏幕型号和驱动协议。

## 当前状态

| 功能 | 确认结果 |
| --- | --- |
| ESP-IDF 5.5.4 构建 | 中文四色版本已通过 |
| Wi-Fi | 已连接热点并获取 IP |
| NTP | 日志确认校时成功 |
| HTTPS 证书校验 | 日志确认通过 |
| Open-Meteo | 当前天气与七天预报已返回 |
| G 型屏幕 | 四色色带诊断图实机显示正常 |
| 中文文字与四色天气排版 | 实机显示正常 |
| 中文天气界面实机显示 | 天气、日期、温度和图标均正常 |
| 深度睡眠与更新计划 | 实机运行正常，按设定时间更新 |
| 联网方式 | 使用 Wi-Fi |

显示端使用 100kHz SPI 和 G 型全刷新时序，中文四色天气显示与休眠更新均已跑通。
