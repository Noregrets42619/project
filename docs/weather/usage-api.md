# 天气屏使用与 Open-Meteo API

这一页记录当前天气屏工程的配置与使用。屏幕接线见 [项目与硬件](index.md)，驱动排查过程见 [复刻日志](porting-log.md)。

## 1. 下载与开发环境

```powershell
git clone https://github.com/Noregrets42619/esp32-e-paper-weatherdisplay.git
cd esp32-e-paper-weatherdisplay
```

工程目标为普通 `esp32`，不是 `esp32p4` 或 `esp32c5`。本机使用 VS Code 的 Espressif IDF 扩展，IDF 目录为 `D:\ESP\espidf\v5.5\esp-idf`，实际版本为 `v5.5.4-299-ge46782886f`。

打开工程目录，运行 `ESP-IDF: Open ESP-IDF Terminal`，首次配置执行：

```powershell
idf.py set-target esp32
idf.py menuconfig
```

## 2. 填写自己的 Wi-Fi 密码

**先在 `WiFi Configuration` 中修改 WiFi SSID 和 WiFi Password，再编译烧录。** 仓库里的 `myssid` 和 `mypassword` 只是占位值，ESP32 使用 2.4GHz 网络。

本地也可以保留 `sdkconfig.wifi.local`：

```ini
CONFIG_ESP_WIFI_SSID="YOUR_2_4GHZ_WIFI_NAME"
CONFIG_ESP_WIFI_PASSWORD="YOUR_WIFI_PASSWORD"
```

这个文件不提交 GitHub。已经生成过 `sdkconfig` 时，以当前配置为准，换网要在 `menuconfig` 中一并修改。

## 3. 地点与时区

`Open-Meteo Weather Configuration` 中的默认配置为：

| 配置 | 默认值 |
| --- | --- |
| 地点名称 | 成都市郫都区 |
| 纬度 | 30.80993 |
| 经度 | 103.88253 |
| API 时区 | Asia/Shanghai |

地点名称只用来绘图；天气查询取决于 WGS84 经纬度。这组坐标代表郫筒一带，不是整个郫都区的平均天气。

`WT32-ETH01 Configuration` 中本地 POSIX 时区是 `CST-8`，也就是 UTC+8。接口返回 Unix 时间后，通过本地时区转换成屏幕上的日期和时间。

改成其他中文地点时，若有字符显示为 `?`，先将字符补到 `main/fonts/characters.txt`，再按仓库 README 的方法重新生成字体。

## 4. Open-Meteo 请求

当前接口为 [Open-Meteo Forecast API](https://open-meteo.com/en/docs)：

```text
GET https://api.open-meteo.com/v1/forecast
```

个人非商业用途使用公开免费接口，不需要注册账号或填写 API Key。商业使用及调用限制见 [官方服务方案](https://open-meteo.com/en/pricing)。

固件实际使用的参数如下：

```text
latitude=30.80993
longitude=103.88253
current=temperature_2m,relative_humidity_2m,pressure_msl,wind_speed_10m,wind_direction_10m,weather_code,is_day
daily=weather_code,temperature_2m_max,temperature_2m_min,precipitation_probability_max
timezone=Asia/Shanghai
forecast_days=7
timeformat=unixtime
wind_speed_unit=ms
temperature_unit=celsius
```

代码在 `components/weather/src/weather.c`。`weather_parse.c` 检查单位、数值范围、七天数组和日期先后关系，再把数据交给界面。

### 数据与单位

| 字段 | 屏幕含义 |
| --- | --- |
| `temperature_2m` | 当前温度，℃ |
| `relative_humidity_2m` | 相对湿度，% |
| `pressure_msl` | 海平面气压，hPa |
| `wind_speed_10m` | 请求 m/s，显示时乘 3.6 变成 km/h |
| `wind_direction_10m` | 角度换算为中文风向 |
| `weather_code` | 天气文字与图标 |
| `is_day` | 晴天的太阳或月亮 |
| `temperature_2m_max/min` | 每天最高温和最低温 |
| `precipitation_probability_max` | 当天最大小时降水概率 |

“当前天气”来自天气模型，不是板上实测。七天的天气代码表示各天最严重的天气情况，见 [Open-Meteo 字段说明](https://open-meteo.com/en/docs)。

“今日最高降水概率”不是当前时刻的概率，也不是降水量。缺失时显示“暂无数据”，不会当成 0%。屏幕底部保留 `数据：Open-Meteo.com` 来源标注，数据采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)。

## 5. 编译与烧录

```powershell
idf.py build
idf.py -p COMx flash
idf.py -p COMx monitor
```

`COMx` 换成当前串口。使用 USB 转 TTL，通过 UART0 烧录，TX/RX 交叉连接并共地，逻辑电平为 3.3V。没有自动下载电路时，将 IO0 接地后复位进入下载模式，写完后断开 IO0 与 GND 并复位。

工程使用 4MB Flash 分区表。首次使用或调整分区时，用 `idf.py flash` 完整写入 bootloader、分区表和应用。

## 6. 正常更新过程

```text
Wi-Fi 获取 IP
    -> NTP 校时
    -> HTTPS 请求 Open-Meteo
    -> 解析当前天气和七天预报
    -> 停止 Wi-Fi
    -> 刷新 G 型四色屏
    -> 屏幕休眠
    -> ESP32 深度睡眠
```

默认北京时间 07:00–22:50 每十分钟更新，夜间等到次日 07:00。Wi-Fi 连接失败后约三小时重试。天气请求失败时保留上次画面。

## 7. 手动刷新

使用一个常开、自复位的轻触按键，一端接 WT32-ETH01 的 **EN**，另一端接 **GND**。按下再松开，ESP32 复位并重新联网更新天气，深度睡眠期间也可以这样操作。

当前程序每次启动都会请求天气，因此不需要另写按键检测。IO0 是下载模式引脚，不用作刷新按键。

## 8. 单屏诊断

在 `WT32-ETH01 Configuration` 中启用 `Screen-only diagnostic (no Wi-Fi, SPI 100 kHz)`，重新编译烧录。此模式只显示测试文字和四色色带，ESP32 保持唤醒，方便测量。

诊断失败后有 60 秒测量窗口，窗口结束时 RST 会被主动拉低。测试完成后关闭诊断模式、重新编译烧录，才能恢复天气功能。
