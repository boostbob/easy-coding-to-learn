由于我们常常在 shell 里工作，启动 vim 编辑的时候，可能还需要做一些其他的工作，但是编辑的工作并没有完成，所以我们其实不想真正的关闭 vim，vim 提供了一些支持。
首先可以使用 :! 来执行 shell 命令，比如编辑的时候突然需要查看一下某个目录下的文件列表，我们可以输入 :!ls -l 试试。
![](http://develop-developer.oss-cn-hangzhou.aliyuncs.com/images/4xRcruFzEdYu3wg7w-v7m0JoiXlrFqQvrVK5x_wB5t.png?x-oss-process=style/txt-water)

回车执行后，vim 显示文件列表，并且提示 "请按 ENTER 或其它命令继续"，按 enter 后又回到了 vim 的界面，并且 vim 里的环境依然保持。
如果我们需要连续的执行多次领命，可能每次去输入 :! 就不是很方便了，我们可以让 vim 直接创建一个 shell，输入 :shell 新开一个 shell 环境，使用完毕后 exit 回到 vim 的环境。
因为 vim 也是一个 linux 命令，所以我们依然可使用 ctrl + z 把它挂起，然后使用 fg 恢复。

```bash
[1]+  Stopped(SIGTSTP)        vim poem.txt
wangbo@wangbo-VirtualBox:~/test/vim$ jobs
[1]+  Stopped(SIGTSTP)        vim poem.txt
wangbo@wangbo-VirtualBox:~/test/vim$ fg
```

如果我们要注销或者关机、重启，仍然想保持 vim 的工作状态怎么办? 这个时候需要把 vim 的工作环境保存成一个文件，使用 :mksession 命令，我们先复制 3 行，然后执行 :mksession sess.vim 和 :q 退出 vim，发现磁盘上多了一个 sess.vim 文件，用 file 甄别说是一个 vim 数据文件。

```bash
wangbo@wangbo-VirtualBox:~/test/vim$ ls -l
total 32
-rw-rw-r-- 1 wangbo wangbo   39 12月 17 14:03 1.txt
-rw-rw-r-- 1 wangbo wangbo   15 12月 17 14:28 2.txt
-rw-rw-r-- 1 wangbo wangbo   23 12月 16 16:43 hello.txt
-rw-rw-r-- 1 wangbo wangbo   58 12月 20 16:30 main.c
-rw-rw-r-- 1 wangbo wangbo  580 12月 20 17:04 poem.txt
-rw-rw-r-- 1 wangbo wangbo   76 12月 18 10:03 print.c
-rw-rw-r-- 1 wangbo wangbo 5043 12月 20 17:23 sess.vim
wangbo@wangbo-VirtualBox:~/test/vim$ file sess.vim
sess.vim: data
```

下次怎么恢复呢? 我们可以直接运行 vim -S sess.vim，也可以 vim 进入后，敲 :source sess.vim 执行一次。
![](http://develop-developer.oss-cn-hangzhou.aliyuncs.com/images/c26HmnWKzPhc5i6s2-uy-h8AQvE1qY8H4gl5o12Yn7.png?x-oss-process=style/txt-water)

还记得我们复制过 3 行文本吗? 进去直接输入 p 命令，发现文本被粘贴上了。说明之前的 vim 工作状态被恢复了。
![](http://develop-developer.oss-cn-hangzhou.aliyuncs.com/images/Js4h3fHydsN8it7Lb-aRnphJwVNlYgyjEpvYuvcoe7.png?x-oss-process=style/txt-water)
