# Git 常用操作

这一页保留原来的 Git 常用命令，并扩写当前项目的源码仓库和静态网页仓库提交流程。

## 原始常用命令

* `git init`  - 初始化git.
* `git add .` - 将所有文件提交至缓冲区.
* `git commit -m "example"` -  编写提交备注.
* `git push origin main` - 将文件提交至仓库main分支.

## 当前仓库地址

源码项目仓库：

```text
git@github.com:Noregrets42619/project.git
```

静态网页仓库：

```text
git@github.com:Noregrets42619/Noregrets42619.github.io.git
```

当前工作区根目录使用项目仓库作为 `origin`。如果需要确认：

```powershell
git remote -v
```

如果需要重新设置：

```powershell
git remote set-url origin git@github.com:Noregrets42619/project.git
```

## 提交文档源码

```powershell
cd "D:\Material for MkDocs"

git status
git add mkdocs.yml docs
git commit -m "Rewrite site for ESP32 Host MIDI piano"
git push origin main
```

## 构建静态网页

```powershell
cd "D:\Material for MkDocs"
mkdocs build
```

构建完成后，静态网页会生成到：

```text
D:\Material for MkDocs\site
```

## 提交静态网页

如果 `site` 是独立 Git 仓库：

```powershell
cd "D:\Material for MkDocs\site"

git remote -v
git remote set-url origin git@github.com:Noregrets42619/Noregrets42619.github.io.git

git status
git add .
git commit -m "Deploy ESP32 Host MIDI piano docs"
git push origin main
```

如果 `site` 不是独立 Git 仓库，而是由根目录项目仓库跟踪，就在根目录一起提交：

```powershell
cd "D:\Material for MkDocs"

git add site
git commit -m "Build static site"
git push origin main
```

## 常见检查

查看当前分支：

```powershell
git branch --show-current
```

查看提交历史：

```powershell
git log --oneline -5
```

查看某个文件的修改：

```powershell
git diff -- docs/index.md
```

撤销暂存但保留文件修改：

```powershell
git restore --staged docs/index.md
```

## 提交建议

- 文档源码和构建产物最好分开提交。
- 如果 `site` 是独立仓库，先提交源码仓库，再进入 `site` 提交网页仓库。
- 每次 `mkdocs build` 后先本地打开或运行 `mkdocs serve` 检查导航。
- 不要把个人密钥、WiFi 密码、真实设备日志中的敏感信息提交到公开仓库。
