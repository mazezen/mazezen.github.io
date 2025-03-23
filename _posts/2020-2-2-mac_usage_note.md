---
layout: post
title: "Mac系统效率开发笔记集合"
date: 2020-2-2
tags: [随笔]
comments: true
author: mazezen
---

## Homebrew
```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## iterm2+OhMyZsh
1. 下载 <a href="https://iterm2.com/downloads.html" target="_blank" rel="noopener">iterm2</a>
2. 解压完移动至 /Applications
3. 安装Oh My Zsh.

在安装 *Oh My Zsh* 之前我们需要将 **Zsh** 设置为当前用户的默认 *Shell*。可以通过以下几个命令查看并配置.

```shell
more /etc/shells 查看当前全部的 Shell
echo $SHELL 查看当前的 Shell
chsh -s /bin/zsh 设置 ZSH 为当前用户的默认 Shell
```

安装好之后，可以在当前用户目录 `~/` 下通过 `ls -a` 查看到新增 `.oh-my-zsh` 文件夹 和 `.zshrc` 文件
`zshrc` 是 **ZSH** 的配置文件 ，如果之前没有此文件，会通过 **Oh My ZSH** 安装自动生成，里面会存放一些环境变量及配置，在终端启动时运行。当然了，也可以配置自定义的环境变量。

```shell
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

5. 配置 oh my zsh

   * 自动补全  zsh-autosuggestions
     ```shell
     git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-$ZSH/custom}/plugins/zsh-autosuggestions 

     vim ~/.zshrc
     plugins=(
        // other plugins...
        zsh-autosuggestions
     )
     ```

   * 命令行关键字高亮显示
     ```shell
     git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
     
     vim ~/.zshrc
     plugins=(
        // other plugins...
        zsh-syntax-highlighting
     )
     ```
   * zsh-vi-mode
     ```shell
     git clone https://github.com/jeffreytse/zsh-vi-mode $ZSH_CUSTOM/plugins/zsh-vi-mode

     vim ~/.zshrc
     plugins=(
        // other plugins...
        zsh-vi-mode
     )
     ```
   * souce ~/.zshrc

   * 重启 iterm2


