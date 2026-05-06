#Building Log

## 📂 MkDocs Commands
* `mkdocs new [Ecoli]` - 新建一个名字为Ecoli的页面
* `mkdocs serve` - 开启自带服务器实时预览【**此版本存在bug，需将click降至8.2.1版本**】.
```test
(pip install click==8.2.1)
```
* `mkdocs build` - 构建网页端.
* `mkdocs -h` - 打印帮助文档.

## 📂 Git Commands
* `git init`  - 初始化git.
* `git add .` - 将所有文件提交至缓冲区.
* `git commit -m "example"` -  编写提交备注.
* `git push origin main` - 将文件提交至仓库main分支.

## 📂 文件结构

为了让 MkDocs 的左侧导航栏逻辑清晰，采用以下目录树组织 `docs` 文件夹：

```text
.
├── mkdocs.yml              # 站点的配置文件（主题、插件、导航）
├── docs/
│   ├── index.md            # 网站首页（上面的介绍放在这）
│   ├── peripherals/        # 外设驱动库
│   │   ├── uart.md         # 串口通信：DMA接收、格式化输出
│   │   ├── i2c.md          # I2C协议：传感器读写示例
│   │   ├── gpio.md         # GPIO：中断优先级、边沿触发
│   │   └── timers.md       # 定时器：PWM输出、输入捕获
│   ├── config-tools/       # MCUXpresso 工具链
│   │   ├── pin-config.md   # 引脚映射与复用技巧
│   │   └── clock-config.md # 时钟树配置（RT1064的灵魂）
│   ├── insights/           # 开发心得
│   │   ├── git-workflow.md # 嵌入式项目的 Git 提交规范
│   │   └── mkdocs-tips.md  # Material 主题的神仙插件配置
│   └── assets/             # 静态资源
│       ├── images/         # 存放原理图、Config Tools 截图
│       └── downloads/      # 存放示例代码的压缩包或 PDF

```

## 💡 建议

* **善用 Admonitions (警告/提示框)**：在 Material 主题中，使用 `!!! info` 或 `??? bug` 这种语法可以让你的函数注意事项非常醒目。
* **代码高亮**：在 `mkdocs.yml` 中开启 `pymdownx.highlight` 插件，这样你的 C 语言和 Git 命令会非常漂亮。

!!! success "装填进度"
    Building my own Blog 装修中....
   
    - [X] 网页框架搭建与发布
    - [ ] 精通 Material for MkDocs
    - [ ] 熟悉 Markdown 语法
    - [ ] 完善内容