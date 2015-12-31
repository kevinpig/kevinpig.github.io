---
layout: post
title: 使用Atom进行Hexo博客的编写
date: '2015-12-30 20:10'
---

## 使用md-write书写静态博客

### 插件安装

  由于Atom插件的仓库在中国经常访问失败，因此我们使用源代码实施进行安装

 1. 安装md-writer插件
```
$ git clone https://github.com/zhuochun/md-writer
$ cd md-writer
$ npm install
 ```
 2. 安装markdown-pdf插件
```
 $ git clone https://github.com/travs/markdown-pdf
 $ cd markdown-pdf
 $ npm install
```
<!-- more -->

### 博客创建

![Create New Post](/images/2015/12/new_post.gif)

### 博客书写

1. Insert Image

 ![Insert Image](/images/2015/12/insert_image.gif)
2. Insert reference link

 ![insert reference](/images/2015/12/insert_link.gif)
3. Insert table

### 博客发布
 在Shell中执行hexo命令来执行博客的发布和Push源文件到github中

## 转化为pdf

 Ctrl+Shift-P 执行 Markdown Pdf:Convert
