---
layout: post
title: Go开发环境搭建
date: 2022-05-10
tags: Go
---

## 安装 [Go](https://golang.google.cn/dl/)

```bash
tar -xf go1.16.4.darwin-amd64.tar.gz -C ~/Applications/
cat ~/.zshrc
GOROOT=/Users/dengyou/Applications/go
PATH=$GOROOT/bin:$GOPATH/bin/:$PATH
```
```
go version
go version go1.16.4 darwin/amd64
```
```
mkdir -pv /Users/dengyou/Desktop/GitHub/LearnGolang/goproject/{src,bin,pkg}
go env -w GOPATH="/Users/dengyou/Desktop/GitHub/LearnGolang/goproject"
go env -w GOBIN="$GOPATH/bin"
go env -w GOPROXY=https://goproxy.cn
go env -w GO111MODULE="auto"
```

## 安装 VSCODE

[vscode官网下载]( https://code.visualstudio.com/)最新版

- **安装 Go 语言扩展**

![](/images/pic/go-extension.png)

- **安装 Go 语言扩展需要的工具集合**

```
// 这些扩展很多被墙了, 需要配置GOPROXY代理才能正常安装:
go env -w GOPROXY=https://goproxy.cn
```

设置变量过后为了保证vscode能正常使用, 请重启vscode

1. 打开命令面板: Shift + Command + P(mac) | Shift + Ctrl + P(windows)
2. 输入: Install/Update 搜索 Go扩展依赖工具安装命令

![](/images/pic/vscode-command.png)

3. 勾选所有选项进行安装

![](/images/pic/vscode-command2.png)

如果失败也可以手动安装

```
go install -v golang.org/x/tools/gopls@latest
go install -v honnef.co/go/tools/cmd/staticcheck@latest
go install -v github.com/go-delve/delve/cmd/dlv@latest
go install -v github.com/haya14busa/goplay/cmd/goplay@latest
go install -v github.com/josharian/impl@latest
go install -v github.com/fatih/gomodifytags@latest
go install -v github.com/cweill/gotests/gotests@latest
go install -v github.com/ramya-rao-a/go-outline@latest
go install -v github.com/uudashr/gopkgs/v2/cmd/gopkgs@latest
```

## VSCODE 插件安装

- 单词拼写检查: Code Spell Checker
- 快捷运行代码的插件: Code Runner
- 最好用Git工具没有之一: Gitlens
- AI 代码生成: Tabnine AI Code
- 设计配色方案: GapStyle VS

## 配置vscode打印run test详情内容

通过修改 go.testFlags 的配置值为: ["-v"], 就开启了test 打印详细日志功能

![](/images/pic/vscode-test-debug.png)

```
{
    "workbench.colorTheme": "One Dark Pro",
    "terminal.integrated.shell.windows": "C:\\Program Files\\Git\\bin\\bash.exe",
    "go.useLanguageServer": true,
    "explorer.confirmDelete": false,
    "explorer.confirmDragAndDrop": false,
    "workbench.iconTheme": "vscode-icons",
    "vsicons.dontShowNewVersionMessage": true,
    "http.proxySupport": "off",
    "go.toolsManagement.autoUpdate": true,
    "terminal.integrated.tabs.enabled": true,
    "terminal.integrated.defaultProfile.windows": "Git Bash",
    "git.autofetch": true,
    "files.autoSave": "afterDelay",
    "go.testFlags": ["-v"]
}
```


