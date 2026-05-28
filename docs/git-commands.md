# Git 常用操作

这一页保留原来写的 Git 常用命令，也记录这个 MkDocs 站点和静态网页仓库的提交方式。

## 原始常用命令

* `git init`  - 初始化git.
* `git add .` - 将所有文件提交至缓冲区.
* `git commit -m "example"` -  编写提交备注.
* `git push origin main` - 将文件提交至仓库main分支.

## 当前两个仓库

源码项目仓库：

```text
git@github.com:Noregrets42619/project.git
```

静态网页仓库：

```text
git@github.com:Noregrets42619/Noregrets42619.github.io.git
```

根目录项目仓库使用：

```powershell
git remote set-url origin git@github.com:Noregrets42619/project.git
```

`site` 静态网页仓库使用：

```powershell
git remote set-url origin git@github.com:Noregrets42619/Noregrets42619.github.io.git
```

## 文档源码提交命令

```powershell
cd "D:\Material for MkDocs"
git status
git add mkdocs.yml docs
git commit -m "Rewrite site as MIDI piano porting log"
git push origin main
```

## 静态网页构建命令

```powershell
cd "D:\Material for MkDocs"
mkdocs build
```

构建结果生成在：

```text
D:\Material for MkDocs\site
```

## 静态网页提交命令

```powershell
cd "D:\Material for MkDocs\site"
git status
git add .
git commit -m "Deploy MIDI piano porting log"
git push origin main
```

之前线上页面出现过 Git 冲突标记，原因是 `site` 仓库里的旧 HTML 带着合并冲突文本被提交到了 GitHub Pages。后来确认本地 `mkdocs serve` 正常，问题出在静态网页仓库里提交了旧构建产物。这个站点后面都按 `mkdocs build` 生成 `site`，再提交 `site` 仓库。
