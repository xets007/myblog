---
title: Vim Tips
date: 2016-05-15 12:05:52
categories:
- Toolbox
tags:
- Vim
---

> Vim作为Linux下必备开发工具，熟练掌握，可以大大提升开发及编辑效率。

> 基本上所有Linux发行版都预装了Vim，开箱即用，唾手可得，无需再下载安装。

> Vim强大的同时也带来了复杂性，学习成本较高，需要慢慢积累，先掌握基础操作，然后再遇到问题解决问题。Vim使用近三年，自己掌握了一些Vim有意思的特性，现在分享出来，希望对你也有帮助。

**Think Vim before you use Emacs.**

<!-- more -->

### 0. tab, window, buffer

![tab-window-buffer](http://7xtc3e.com1.z0.glb.clouddn.com/vim-tips/tabs-windows-buffers.png)

An illustration of the relationships among Vim's tabs, windows, and buffers. Buffers 4 and 6 are considered hidden, since they are not visible in any window.

### 1. command-line window

In the command-line window the command line can be edited just like editing in any window.

To open command-line window, From Normal mode, use the `"q:"`, `"q/"` or `"q?"` command.

![command line window](http://7xtc3e.com1.z0.glb.clouddn.com/vim-tips/command-line-window.png)

### 2. edit binary file

The following command replaces the buffer with a hex dump:

```vimscript
:%!xxd
```

You can edit the hex bytes, then convert the file back to binary with the command:

```vimscript
:%!xxd -r
```

The above command reverses the hex dump by converting the hex bytes to binary (the printable text in the right column is ignored).

![edit-binary-1](http://7xtc3e.com1.z0.glb.clouddn.com/vim-tips/edit-binary-1.png)

> - %  current file name
> - %< current file name without extension

```vimscript
:%！hexdump -C
```

![edit-binary-2](http://7xtc3e.com1.z0.glb.clouddn.com/vim-tips/edit-binary-2.jpg)

```vimscript
:%!xxd -p
```

![edit-binary-3](http://7xtc3e.com1.z0.glb.clouddn.com/vim-tips/edit-binary-3.png)

```vimscript
:%!xxd -p -r
```

![edit-binary-4](http://7xtc3e.com1.z0.glb.clouddn.com/vim-tips/edit-binary-4.png)

> - -p    -plain
> - -r    -revert

### 3. indent

```
gg=G
```

In normal mode, typing `gg=G` will reindent the entire file. This is a special case, `=` is an operator. Just like `d` or `y`, it will act on any text that you move over with a cursor motion command. In this case, `gg` positions the cursor on the first line, then `=G` re-indents from the current cursor position to the end of the buffer.

> - \>\> Indent line by shiftwidth spaces
> - << De-indent line by shiftwidth spaces
> - 5>> Indent 5 lines
> - 5== Re-indent 5 lines

### 4. multiline editing

In visual block mode, you can press `I` to insert text at the same position in multiple lines, and you can press `A` to append text to each line in a block.

Insert

> 1. Use Ctrl+V(or Ctrl-Q if you use Ctrl-V for paste) to select the column of text in the lines you want to insert.
> 2. Then hit I and type the text you want to insert.
> 3. Then hit Esc, wait 1 second and the inserted text will appear on every line.

Append

> 1. Use Ctrl+V(or Ctrl-Q if you use Ctrl-V for paste) to select the column of text in the lines you want to append.
> 2. Press $ to extend the visual block to the end of each line.
> 3. Then hit A and type the text you want to append.
> 4. Then hit Esc, wait 1 second and the inserted text will appear on every line.

### 5. window resize

> - equal size        ^W=
> - max height        ^W_
> - max width         ^W|

### 6. list mode

`:set list` displays whitespace, `:set nolist` displays normally.

`set listchars`: Strings to use in `list` mode.


### 7. digraphs

Digraphs are used to enter characters that normally cannot be entered by an ordinary keyboard.

`:dig[raphs]` show currently defined digraphs.

![digraphs](http://7xtc3e.com1.z0.glb.clouddn.com/vim-tips/digraphs.png)

In insert mode, `CTRL-K {char1} {char2}` to enter digraphs.

### 8. diff mode

```bash
$ vimdiff file1 file2 [file3 [file4]]
$ vim -d file1 file2 [file3 [file4]]
```

![diff-mode](http://7xtc3e.com1.z0.glb.clouddn.com/vim-tips/diff-mode.jpg)

> - do - Get changes from other window into the current window.
> - dp - Put the changes from current window into the other window.
> - ]c - Jump to the next change.
> - [c - Jump to the previous change.
> - zo - open fold.
> - zc - close fold.
> - :diffu[pdate] - Update the diff highlighting and folds.
> - :qa - quit all
> - :wa - write all
> - :wqa - write, then quit all
> - :qa! - force to quit all

### 9. edit compressed files

To see more info:

```
:h gzip    (*.gz, *.bz2, *.Z)
:h zip     (*.zip)
:h tar     (*.tar)
```

### 10. substitute

`:%s/foo/bar/g`
    Find each occurrence of 'foo' (in all lines), and replace it with 'bar'.

`:s/foo/bar/g`
    Find each occurrence of 'foo' (in the current line only), and replace it with 'bar'.

`:%s/foo/bar/gc`
    Change each 'foo' to 'bar', but ask for confirmation first.

`:5,12s/foo/bar/g`
    Change each 'foo' to 'bar' for all lines from line 5 to line 12 (inclusive).

`:%s/\s\+$//`
    Delete all trailing whitespace (at the end of each line).
    In a search, \s finds whitespace (a space or a tab), and \+ finds one or more occurrences.

### 11. recording a macro

Each register is identified by a letter `a` to `z`.

To enter a macro, type:

```
q<letter><commands>q
```

To execute the macro <number> times (once by default), type:

```
<number>@<letter>
```

So, the complete process looks like:

> 1. qd  start recording to register d
> 2. ... your complex series of commands
> 3. q   stop recording
> 4. @d  execute your macro
> 5. @@  execute your macro again

### 12. count

```
g CTRL-G            (:h g_CTRL-G word-count byte-count)
```

![count-1](http://7xtc3e.com1.z0.glb.clouddn.com/vim-tips/count-1.png)

```
{Visual}g CTRL-G    (:h v_g_CTRL-G)
```

![count-2](http://7xtc3e.com1.z0.glb.clouddn.com/vim-tips/count-2.png)

### 13. changing tabs

```
:retab

:[range]ret[ab][!] [new_tabstop]
```

Replace all sequences of white-space containing a <Tab> with new strings of white-space using the new tabstop value given.  If you do not specify a new tabstop size or it is zero, Vim uses the current value of 'tabstop'.

### 14. su-write

If you find you do not have permission to perform :w, use the following:

```
:w !sudo tee % > /dev/null
```

### 15. open file with specified line number

```
$ vim filename +10
```

```
   +                    Start at end of file
   +<lnum>              Start at line <lnum>
   --cmd <command>      Execute <command> before loading any vimrc file
   -c <command>         Execute <command> after loading the first file
```

### 16. jump

In normal mode, `%` jumps to corresponding item, e.g. from an open brace to its matching closing brace.

Like a web browser, you can go back, then forward:

- Press Ctrl-O to jump back to the previous (older) location.
- Press Ctrl-I (same as Tab) to jump forward to the next (newer) location.

Display the jump list for the current window with:

```
:jumps
```

### 17. format json

```
:%!python -m json.tool
```

### 18. vim plugin manager

- [What is the difference between the vim package managers?](http://vi.stackexchange.com/questions/388/what-is-the-difference-between-the-vim-package-managers)
- [Why I switched from Vundle to Plug](https://jordaneldredge.com/blog/why-i-switched-from-vundle-to-plug/)

### 19. useful resources

- [Vim Tips Wiki](http://vim.wikia.com/wiki/Vim_Tips_Wiki)
- [Vim Awesome](http://vimawesome.com/)
