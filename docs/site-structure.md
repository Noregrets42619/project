# Ecoli 站点记录

站点继续使用 Material for MkDocs，保留 Ecoli 的名字、图标与原来的主题。文档按开发板和工程分组，P4C5 MIDI Piano 与 WT32-ETH01 天气屏分别放在导航中。

## MkDocs 命令

```powershell
mkdocs serve
mkdocs build
mkdocs build --strict
```

`serve` 用于本地预览，`build` 生成静态网页，`--strict` 用来检查文档构建中的警告。此前 `serve` 遇到过 click 版本兼容问题，当时通过 `pip install click==8.2.1` 处理。

## 当前文档树

```text
.
├── mkdocs.yml
├── docs/
│   ├── index.md                  # 两个项目的首页入口
│   ├── piano/index.md            # 原 MIDI Piano 概览
│   ├── usage-guide.md            # MIDI Piano 使用说明
│   ├── project-setup.md          # P4C5 工程准备
│   ├── usb-display.md            # USB MIDI 与 SPI 显示
│   ├── audio-network-ble.md      # 音频、WiFi 与 BLE
│   ├── issues-and-fixes.md       # MIDI Piano 报错处理
│   ├── project-status.md         # MIDI Piano 状态
│   ├── weather/
│   │   ├── index.md              # 天气屏项目与硬件
│   │   ├── porting-log.md        # 天气屏复刻日志
│   │   ├── usage-api.md          # 使用与 Open-Meteo API
│   │   └── issues-and-status.md  # 天气屏报错与状态
│   ├── site-structure.md
│   ├── git-commands.md
│   ├── img/
│   │   ├── usage/                # MIDI 使用截图
│   │   └── weather/              # 天气界面像素预览
│   └── javascripts/mathjax.js
└── site/                         # 静态网页，有独立 Git 仓库
```

MIDI Piano 的旧页面地址继续保留。原首页内容放到 `piano/index.md`，首页增加两个项目入口。`new.md` 是旧的完整记录，仍不加入正式导航。

## 文档与网页仓库

文档源码放在 [project](https://github.com/Noregrets42619/project)，生成网页放在 [Noregrets42619.github.io](https://github.com/Noregrets42619/Noregrets42619.github.io)。`site` 目录不再由文档源码仓库重复跟踪，网页更新在它自己的仓库中提交。

天气屏固件与 README 放在独立的 [esp32-e-paper-weatherdisplay](https://github.com/Noregrets42619/esp32-e-paper-weatherdisplay) 仓库。
