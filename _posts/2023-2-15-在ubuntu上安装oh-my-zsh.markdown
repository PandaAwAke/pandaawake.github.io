---
layout: post
title:  "在ubuntu上安装oh-my-zsh"
date:   2022-02-15 12:00:00 +0800
categories: Ubuntu Oh-my-zsh
---



## 目标

在 Ubuntu 安装 Oh-my-zsh 和一些我用的插件。

参考：[oh-my-zsh 国内安装及配置 - 掘金 (juejin.cn)](https://juejin.cn/post/7023578642156355592)



## 安装 oh-my-zsh

### 安装 zsh

```bash
sudo apt update
sudo apt install -y zsh
```

### 安装 oh-my-zsh

```bash
wget https://gitee.com/mirrors/oh-my-zsh/raw/master/tools/install.sh
chmod +x install.sh
sudo rm -rf ~/.oh-my-zsh
./install.sh
```

> 搬运自原帖：
>
> 如果发现很慢，可以修改为`gitee`：
>  `vim install.sh`进入编辑状态：
>  找到以下部分：
>
> ```cmd
> # Default settings
> ZSH=${ZSH:-~/.oh-my-zsh}
> REPO=${REPO:-ohmyzsh/ohmyzsh}
> REMOTE=${REMOTE:-https://github.com/${REPO}.git}
> BRANCH=${BRANCH:-master}
> ```
>
> 然后将中间两行改为：
>
> ```cmd
> REPO=${REPO:-mirrors/oh-my-zsh}
> REMOTE=${REMOTE:-https://gitee.com/${REPO}.git}
> ```
>
> 然后保存退出：`:wq`
> 重新执行即可。



## 配置文件和插件

### 配置 zsh

```bash
vim ~/.zshrc	# 你可以用别的编辑器
```

我自己的配置文件（去掉注释）如下：

```bash
ZSH_THEME="af-magic"

plugins=(
    git
    zsh-autosuggestions
    zsh-syntax-highlighting
    git-open
    z
    sudo
    zsh-completions
)

fpath+=${ZSH_CUSTOM:-${ZSH:-~/.oh-my-zsh}/custom}/plugins/zsh-completions/src

source $ZSH/oh-my-zsh.sh

setopt no_nomatch
```



### 安装插件

我上面配置文件已经写了很多插件了，不过并不是所有插件都要安装，需要装的插件分别安装如下：

### zsh-autosuggestions

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```



### zsh-syntax-highlighting

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```



### git-open

```bash
git clone https://github.com/paulirish/git-open.git $ZSH_CUSTOM/plugins/git-open
```



### zsh-completions

```bash
git clone https://github.com/zsh-users/zsh-completions ${ZSH_CUSTOM:-${ZSH:-~/.oh-my-zsh}/custom}/plugins/zsh-completions
```

