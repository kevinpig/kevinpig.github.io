title: emacs 常用技巧
date: 2015-08-27 17:25:42
tags:
 - emacs
---

## emacs 查询帮助的方法

学习，从动手开始；高手，从学会学习开始。

### 查询当前模式下有哪些按键绑定

1. C-h m 可以详细的显示当前模式下有哪些按键绑定
2. C-x C-h 查询所有以C-x开头的按键绑定
3. C-c C-h 查询所有以C-c开口的按键绑定

### 查询特定的按键绑定

1. C-h c 提示用户输入一个按键组合，给出这个组合绑定了什么命令
2. C-h k 同上，但是提示信息更加的详细

> 查询C-h c 绑定了什么命名 C-h c C-h c 显示绑定了 describe-key-briefly
> 如果启用的Viper mode，C-h可能被绑定到另外的键，这时候可以使用F1来代替C-h

### 查询某个命令的绑定按键

1. C-h w 按键组合 或者 M-x where-is
> 例如查询where is绑定了什么按键可以 C-h w where-is

<!--more-->

## 文本基本操作

### Copy & Paste （复制粘贴）
C-w 或者 S-delete 删除整行或者整块标记的内容 (whole-line-or-region-kill-region)
M-w 或者 C-insert 绑定到复制(whole-line-or-region-kill-ring-save)
C-y 或者 S-insert 绑定到粘贴(yank or whole-line-or-region-yank)
### Find （查找替换）
查询的命令有search-forward (isearch-forward) 和search-backword（isearch-backwark），可以使用C-h c查询绑定的按键
按C-s 开始输入需要查找的字符串，C-s向前搜索下一个，C-r向后搜索下一个
### Delete （删除）
没有按键绑定 删除整行或者整块标记的内容 (whole-line-or-region-delete)
### Move （移动）
1. C-a 移动到行首
2. C-e 移动到行尾## 标题 ##
3. C-p Previous-line
4. C-n Nex-Line
5. C-b 向后一个字符(backward)
6. C-f 向前一个字符(forward)
7. M-b 向后一个word
8. M-f 向前一个word
9. M-< 文档最前
10. M-> 文档最后

## emacs 中启动多个shell
> C-u M-x eshell #默认情况如果启动shell在输入eshell将jump到已有的shell，如果想新启动一个shell在前面增加C-u的前缀

## emacs 包管理器

### 安装插件
> M-x package-install

### 插件列表
> M-x package-list-packages


## 使用Tramp编辑远程文件
有时候我们希望可以使用本地emacs来编辑修改远程的文件，而不需要在远程的host机器安装任何的东西。Tramp默认使用ssh打开远程的文件

> 使用`ido-find-file` @xxx/127.0.0.1:/path/to/file 来打开编辑远程文件

> 也可以使用 `ido-dired` @xxx/127.0.0.1:/path/to/dir 开个一个远程的dired，这样通过dired打开的文件也可以通过tramp来编辑

## dired -- emacs 内嵌的资源管理器

C-x D(ido-dired) 打开一个dired buffer
常用操作
 * 预览 v -- 在dired选中文件按v即可预览文件，按q即可退出
 * 上一行 p 下一行 n
 * q 或 ^ 查看当前文件(夹)的父文件夹
 * o or C-o 在另外一个buffer中打开文件
