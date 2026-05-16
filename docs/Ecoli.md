# Ecoli 站点与文档结构

这一页保留 Ecoli 这个站点入口，同时把原来零散的建站记录改成当前 ESP32_Host_MIDI 复刻文档的结构说明。

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
├── mkdocs.yml              # Material for MkDocs 配置
├── docs/
│   ├── index.md            # 首页：项目目标和阅读顺序
│   ├── replication.md      # 复刻步骤：从工程清理到 BLE MIDI
│   ├── configuration.md    # 依赖、组件目录、menuconfig 配置
│   ├── test.md             # 移植日志：按阶段记录完整过程
│   ├── test1.md            # 调试记录：问题、现象和处理办法
│   ├── report.md           # 功能状态与复盘
│   ├── git.md              # Git 常用操作和发布流程
│   ├── img/
│   │   └── favicon.png     # Ecoli 图标
│   └── javascripts/
│       └── mathjax.js      # MathJax 配置
└── site/                   # mkdocs build 生成的静态网页
```

## 内容边界

当前站点只围绕 WT99P4C5-S1 MIDI Piano 迁移展开。旧的练习内容和非主线记录不再作为正式页面维护。

## 编写原则

- 先写能复现的步骤，再写原因解释。
- 所有硬件相关差异集中到配置章节。
- 对于已经踩过的坑，保留日志现象和最终处理方式。
- 对上游 Arduino 代码尽量少改，优先通过 IDF wrapper、薄封装层和配置项适配。
