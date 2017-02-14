---
title: Linux开发四件利器
date: 2017-02-14 21:59:57
categories: Toolbox
tags:
- Vim
- Tmux
- Git
- Zsh
---

人与计算机交互，常见的方式有GUI（Graph User Interface，图形界面）、CLI（Command Line Interface，命令行）、TUI（Touch User Interface，触屏）等。交互方式也会影响软件开发，比如在Windows下，就可以使用强大的IDE --- Visual Studio，而大多数Linux开发环境只提供了命令行界面，只能用命令行工具进行开发。

虽然命令行的学习成本比图形界面高，但熟练运用命令行后开发效率一点不比图形界面低，而且没有图形界面的干扰，更容易抓住技术的本质。下面就介绍Linux开发的四件利器，几乎是做Linux开发绕不开的工具。

<!-- more -->

## Vim

Linux下的编辑器，[Vim](http://www.vim.org/)或[Emacs](https://www.gnu.org/software/emacs/)二选一，我选了普及率更高的Vim。Vim的学习曲线比较陡峭，但只要稍微克服下刚开始先入为主的抵触与害怕，你慢慢会发现Vim越用越有意思，越来越离不开Vim。如果你之前一直用的是图形界面的编辑器，用完Vim会对编辑器有个全新的认识，对文本的结构也会有全新认识，从文本到段落到句子到单词到字符，Vim能做到精确掌控。

![vim](http://7xtc3e.com1.z0.glb.clouddn.com/four-linux-development-tools/vim.png)

Vim的强大离不开灵活的配置及丰富的扩展，你可以将Vim配置成你喜欢的样子，我的Vim配置已放在GitHub，[dotfiles/vim/vimrc](https://github.com/consen/dotfiles/blob/master/vim/vimrc)。从配置可以看出，对于扩展插件管理我用的是[vim-plug](https://github.com/junegunn/vim-plug/)，几个比较有意思的扩展：

- [vim-multiple-cursord](https://github.com/terryma/vim-multiple-cursors)
- [vim-stripper](https://github.com/itspriddle/vim-stripper)
- [vim-gitgutter](https://github.com/airblade/vim-gitgutter)
- [rainbow_parentheses](https://github.com/kien/rainbow_parentheses.vim)
- [tabular](https://github.com/godlygeek/tabular)

你也可以在[Vim Awesome](http://vimawesome.com/)查看各插件的排名。

Vim的各种使用技巧需要慢慢积累，如我之前的博客[《Vim Tips》](https://consen.github.io/2016/05/15/vim-tips/)，几个比较精华的Vim学习资源：

- [A Byte of Vim](https://vim.swaroopch.com/)
- [Vim Tips Wiki](http://vim.wikia.com/wiki/Vim_Tips_Wiki)
- [Vim Casts](http://vimcasts.org/)

## Tmux

如果你是远程连接到Linux服务器进行开发，一定遇到过下面几个问题：

1. 有时可能会同时进行多项操作，不得不建立多个远程连接。
2. 如果网络出现抖动，连接突然中断，又得重新建立连接，重新恢复之前的操作。
3. 想在一个屏幕上不切换窗口，看到所有窗口的输出。

[Tmux](http://tmux.github.io/)（terminal multiplexer）很好地解决了上面的问题，只要建立一个远程连接，就可以开启多个终端，同时进行操作，一个查看文档，一个写代码，一个编译，一个运行程序。

我的Tmux配置，[dotfiles/tmux/tmux.conf](https://github.com/consen/dotfiles/blob/master/tmux/tmux.conf)，Tmux默认命令前置操作为'Ctrl+B'，我改成了'Ctrl+A'。

![tmux](http://7xtc3e.com1.z0.glb.clouddn.com/four-linux-development-tools/tmux.png)

常用的几个tmux命令：

- 会话（session）

```
$ tmux new -s session-name          ; 新建会话
$ tmux detach                       ; 断开会话
$ tmux attach -t session-name       ; 连接会话，远程连接突然中断，再次连接后，attach到原先的会话，继续之前的操作
```

- 窗口（window）

```
Ctrl+A c            ; 新建窗口
Ctrl+A [0-9]        ; 切换窗口
```

- 面板（pane）

```
Ctrl+A -    ; 建立垂直面板
Ctrl+A |    ; 建立水平面板
Ctrl+A z    ; 最大化面板
```

## Git

现在对代码的管理，已是[Git](https://git-scm.com/)一统天下，又有[GitHub](https://github.com)加持，实在是解放了程序员，大大提升开发效率。

Git的命令很多，但日常用的20%都不到：

```
$ git clone
$ git pull
$ git status
$ git diff
$ git add
$ git commit -m ''
$ git push
$ git log

$ git branch
$ git checkout
$ git merge
```

## Zsh

不推荐直接上手Zsh, 先从Bash玩起，因为你可以在开发环境下自己装个Zsh, 但是产品环境并不一定有Zsh，却一定有Bash。先从Bash开始搞懂Linux Shell的核心思想，比如至少对以下几点比较熟练：

- 常用命令
- Bash配置
- 常用快捷键
- Bash脚本编写

之后为了提高shell操作效率，就可以使用[Zsh](http://www.zsh.org/)了，当然离不开Zsh配置管理框架[Oh My Zsh](http://ohmyz.sh/)。

![zsh](http://7xtc3e.com1.z0.glb.clouddn.com/four-linux-development-tools/zsh.png)

我的Zsh配置，[dotfiles/zsh/zshrc](https://github.com/consen/dotfiles/blob/master/zsh/zshrc)。有个扩展鼎力推荐，它使Zsh的使用效率大幅提升，[zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions)。
