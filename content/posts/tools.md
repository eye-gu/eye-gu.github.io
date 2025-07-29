---
title: tools
subtitle:
date: 2024-10-30T10:45:04+08:00
slug: cc1b307
draft: false
author:
  name:
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: false
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---



## mac

[精品MAC应用分享](https://xclient.info/)

[Mac618 - Mac资源免费分享站](https://www.mac618.com/)

[Mac618 - Mac资源免费分享站](https://macked.app/)



## iterm2

oh-my-zsh

zsh-autosuggestions

zsh-syntax-highlighting

sdkman

## sublime

### 默认打开新tab

```json
{
	"open_files_in_new_window": false,	
	"font_size": 14,
	"tab_completion": false
}
```

### 快捷键

```json
[
        { "keys": ["super+r"], "command": "show_panel", "args": {"panel": "replace", "reverse": false} },
        { "keys": ["home"], "command": "move_to", "args": {"to": "hardbol", "extend": false} },
        { "keys": ["end"], "command": "move_to", "args": {"to": "hardeol", "extend": false} },
        { "keys": ["shift+home"], "command": "move_to", "args": {"to": "hardbol", "extend": true}},
        { "keys": ["shift+end"], "command": "move_to", "args": {"to": "hardeol", "extend": true} }
]
```

## idea

<https://jetbra.in/5d84466e31722979266057664941a71893322460>

### plugin

GenerateAllSetter

MapStruct Support

Maven Helper

MybatisX

MybatisCodeHelper

Translation

Checkstyle

SonarLint

arthas

jrebel :[【后端】JRebel 2024.1.2本地激活（适用于Windows & macOS） - 双份浓缩馥芮白 - 博客园](https://www.cnblogs.com/Flat-White/p/18090651)

### 多行tab

Editor -> General -> Editor Tabs -> multiple rows/Tab limit

### 注释

Settings -> Code Style -> Java ，在右边选择 “Code Generation” Tab，然后找到 Comment Code 那块，去掉Line comment at first column，Block comment at first column前面的复选框。
勾上Add a space as line comment start

### endpoins

advanced settings -> search everywhere -> show endpoints tab for projects

### git

advanced settings -> version control -> use modal commit interface for git and mercurial

### import

Editor > Code Style > Java > Scheme Default > Imports

Class count to use import with

Names count to use [static](https://so.csdn.net/so/search?q=static&spm=1001.2101.3001.7020) import with

将这两个数字改大, 防止import *



## 右键专业助手



## clipboard/Paste bot

<https://github.com/Clipy/Clipy>



## scroll reverser



## copyq



## typora

[Typora破解2025最新版破解教程1.10.8_typora 1.10.8-CSDN博客](https://blog.csdn.net/zimumumua/article/details/146481724)



## powershell 7

```
# Set-PSReadLineOption -PredictionViewStyle ListView
Set-PSReadLineOption -PredictionViewStyle InlineView
Set-PSReadLineOption -HistorySearchCursorMovesToEnd
Set-PSReadLineKeyHandler -Key UpArrow -Function HistorySearchBackward
Set-PSReadLineKeyHandler -Key DownArrow -Function HistorySearchForward
if (Test-Path "C:\Users\EDY\.jabba\jabba.ps1") { . "C:\Users\EDY\.jabba\jabba.ps1" }
```

jabba



## jprofile

[reajason.me/writing/jprofilerv15crackedwithida/](https://reajason.me/writing/jprofilerv15crackedwithida/)



## charles

[Charles破解工具](https://www.zzzmode.com/mytools/charles/)



## termius

[Termius PRO 最新学习版-CSDN博客](https://blog.csdn.net/weixin_70488292/article/details/143803402)



## vscode

window.restoreWindows 重新打开上次的窗口

files.readonlyInclude 配置只读文件夹



## git

```shell
# Windows
git config --global core.autocrlf input

# mac
git config --global core.autocrlf true
```

