---
title: ubuntu搭建TexLive环境写slide
date: 2018-05-08 07:03:40
updated: 2018-05-08 07:03:40
tags: tex
categories: tex
keywords: texlive texstudio beamer slide
copyright: true
---

### 前言

* 想做ppt，但几次ppt演示时出现兼容问题，于是想尝试格式稳定的`beamer` 做`ppt` 。
* 本机空间不足，`apt-get install` 命令无法指定安装路径，于是挂载了一个硬盘，手动下载后安装，空间充足可`sudo apt install texlive-full texstudio` 一行命令解决。

### texlive安装

1. 进入准备安装路径，使用更快的清华大学镜像源下载`texlive` 安装包：

<!-- more -->

```
$ cd /media/xsyin/extra
$ wget https://mirrors.tuna.tsinghua.edu.cn/CTAN/systems/texlive/Images/texlive2018.iso
```

2. 单击挂载光盘，用浏览器打开主目录下的`index.html` ，找到`tex Live安装指南` ，或阅读[网络版TEX Live安装指南](http://tug.org/texlive/doc/texlive-zh-cn/texlive-zh-cn.pdf) 。使用图形化安装，安装依赖：

```
$ sudo apt install perl-tk
$ ./install-tl -gui=perltk
```

![install-lnx-main](install-lnx-main.png)

`Installation collections --> change` 选择安装模块，去除一些用不着的语言`Cyrillic` 至`Spanish` ，选上`US and UK English` ：

![stdcoll](stdcoll.png)

`TEXDIR -->change` 修改为安装路径：`/media/xsyin/extra/texlive2018` ，其余默认，开始安装。

3. 配置环境变量，编辑`~/.bashrc`， 添加：

```
export PATH=$PATH:/media/xsyin/extra/texlive2018/bin/x86_64-linux
export MANPATH=/media/xsyin/extra/texlive2018/texmf-dist/doc/man:$MANPATH
export INFOPATH=/media/xsyin/extra/texlive2018/texmf-dist/doc/info:$INFOPATH
```

`source ~/.bashrc` 启用配置。

4. 管理你的安装`tlmgr` ，设置清华大学开源仓库并更新，已安装`beamer`：

```  
$ tlmgr option repository https://mirrors.tuna.tsinghua.edu.cn/CTAN/systems/texlive/tlnet
$ tlmgr update -all
$ tlmgr info beamer
```

也可以`GUI` 模式启动： `tlmgr -gui`

5. 测试安装是否成功：

```
$ tex --version
$ latex sample2e.tex
$ pdflatex sample2e.tex
```

----

### texstudio安装

* texstudio 占用空间不大，使用`sudo apt install texstusio` 安装或在[源码包下载页面](http://texstudio.sourceforge.net/ ) 选择对应的安装包，依赖不满足使用`sudo apt -f install` 安装。
*  `texstudio` 打开，配置语言为中文.

### beamer

* 快速上手一般采用现成的模板，在texstudio中新建，复制以下最小模板，构建并查看：

```
%!TEX program = xelatex

\documentclass{beamer}
\usepackage{xeCJK}

\usetheme{Madrid}

\title[标题]{完整标题}
\author[作者]{完整作者}
\institute[单位]{完整单位列表}
\date[日期]{完整日期}

\begin{document}

%------------------------------------------------

\begin{frame}
    \titlepage
\end{frame}

%------------------------------------------------

\begin{frame}{Outline}
    \tableofcontents
\end{frame}

%------------------------------------------------

\section{背景介绍}

\begin{frame}
\frametitle{中文}

\begin{block}{模块}
内容
	\begin{itemize}
	    \item 条目1
	\end{itemize}
\end{block}

\end{frame}

%------------------------------------------------

\end{document}
```

好了，现在可以开(ku) 心(bi)的写`slide` 了。



### 参考资料

[Beamer 显示中文的模板](http://zhenruichen.com/2015/09/02/Beamer-Chinese.html)