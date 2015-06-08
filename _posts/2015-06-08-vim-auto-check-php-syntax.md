---
layout: post
title: "Vim中添加对PHP的语法检查"
description: ""
category: 
tags: [Vim]
---
平时在 Vim 中敲代码，难免会出现一些低级的语法错误。如果没有安装 [Syntastic](https://github.com/scrooloose/syntastic) 这类语法检查的插件，在页面刷新或者运行脚本报错后才能发现错误，稍显麻烦。   
能不能在写文件时就自动完成语法检查呢？  
很多脚本语言都提供语法检查的命令，比如 php -l，ruby -c。以 PHP 为例，在自己的 ~/.vimrc 中添加简单的 Vimscript，即可实现需要的功能。

```vim
function! CompilePHP()
    let file = shellescape(expand('%:p'))
    let command = "!php -l " . file
    execute command
endfunction
autocmd BufWritePost *.php call CompilePHP()
```
上面的 CompilePHP 函数首先获取文件的完整路径，拼成需要执行的命令，然后执行。在 Vim 中，感叹号 ! 放在最前面，表示执行外部命令。可以在 Command-line 模式下输入 !pwd 进行试验。

```vim
:!pwd
```  
我们会看到 Vim 调用 shell 命令 pwd 的输出结果，再按一下 Enter 键会返回 Vim。```!php -l . file``` 就是调用 php -l 命令对文件进行检查。 

```vim 
autocmd BufWritePost *.php call CompilePHP()  
```
该句配置了一个 autocmd。Vim 中通过 autocmd 定义一个或多个命令，这些命令针对匹配的文件，在触发指定事件后会被执行。这条 autocmd 表示在 Buffer 中的数据写入php文件后，调用 CompilePHP() 函数。
这样在 Vim 中编辑 PHP 文件时，每次按下 :w 后都会自动进行语法检查。美中不足的是无论是否出现语法错误，每次写文件后都会出现下面的提示。

```
Press ENTER or type command to continue	
```
是否能在语法正确的时候，不提示上述消息呢？我们可以对前面的脚本进行升级。

```vim
function! CompilePHP()
    let file = shellescape(expand('%:p'))
    let command = "silent !php -l " . file
    execute command
    if v:shell_error != 0
        silent !clear
        let command = "!php -l " . file
        execute command
    else
        redraw!
    endif
endfunction
autocmd BufWritePost *.php call CompilePHP()
```
代码逻辑是先运行 silent !php -l 命令，如果返回值不为 0 则再次运行该命令，输出具体的错误信息。
如果希望 Vim 在执行外部命令后不出现 Press ENTER or type command to continue 的提示，可以通过  :silent ! 来执行命令。尝试运行下面的命令：

```vim
:silent !echo Hello, world.
```
我们将不会看到任何输出。  
silent !clear 命令帮助我们只看到下一个运行命令的输出。  
还有一点需要注意的是如果是在终端运行 Vim，在运行完 :silent ! 后需要通过 :redraw! 来修正屏幕输出。GUI Vim 比如 Mac Vim 或者 gVim 则可以不加 redraw! 语句。
###参考文章
[http://learnvimscriptthehardway.stevelosh.com/chapters/52.html](http://learnvimscriptthehardway.stevelosh.com/chapters/52.html)

