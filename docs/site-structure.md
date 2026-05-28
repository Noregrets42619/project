# Ecoli 站点记录

这一页记录 MkDocs 站点当前结构。站点名字、图标和 Ecoli 这个自定义入口继续保留。

## MkDocs Commands

* `mkdocs new [Ecoli]` - 新建一个名字为Ecoli的页面
* `mkdocs serve` - 开启自带服务器实时预览【**此版本存在bug，需将click降至8.2.1版本**】.
```test
(pip install click==8.2.1)
```
* `mkdocs build` - 构建网页端.
* `mkdocs -h` - 打印帮助文档.

## 当前文档树

```text
.
├── mkdocs.yml                    # Material for MkDocs 配置
├── docs/
│   ├── index.md                  # 首页
│   ├── project-setup.md          # 工程准备与接入记录
│   ├── usb-display.md            # USB MIDI 与 SPI 显示移植记录
│   ├── audio-network-ble.md      # 音频、WiFi 与 BLE 移植记录
│   ├── issues-and-fixes.md       # 报错与处理记录
│   ├── project-status.md         # 项目状态
│   ├── site-structure.md         # Ecoli 站点记录
│   ├── git-commands.md           # Git 常用操作
│   ├── img/
│   │   └── favicon.png           # Ecoli 图标
│   └── javascripts/
│       └── mathjax.js            # MathJax 配置
└── site/                         # mkdocs build 生成的静态网页
```

## 内容边界

这个站点现在只记录 WT99P4C5-S1 MIDI Piano 移植过程。旧的练习内容和非主线记录没有放进正式导航。
