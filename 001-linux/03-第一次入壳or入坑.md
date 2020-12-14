Shell 英文原意是壳ké(坑kēng)，可以简单的理解成用户和系统的翻译官，Shell 接受你的输入传给操作系统执行，并且回显执行的结果，所以 Shell 是操作系统提供给用户的一个接口，可以运行命令、解释程序等。Shell 解释器有很多种，通常我们见到的是 Bash，还有其他的例如 zsh、ksh、tcsh、csh、dash 等，不需要去学习，注意是不需要。

在电影创战纪中sam进入废弃的地下室，在电脑上为了弄清楚是谁登录了系统(他想确认是不是他的father)，输入的第一个命令就是 whoami，也就是让 Shell 告诉我们，当前登录的账号名称，如果还想知道还有谁登录了系统可以输入 w 命令(该命令还显示了系统运行时间和负荷，把 uptime 命令的事情干了)。

```
~ 🍎 whoami
wangbo
~ 🍎 w
15:36  up 15 days, 43 mins, 5 users, load averages: 3.02 3.32 3.18
USER     TTY      FROM              LOGIN@  IDLE WHAT
wangbo   console  -                241120  15days -
wangbo   s000     -                14:52       - w
```

输入 uname 可以显示系统的名称，如果想得到完整的系统的信息输入 uname -a，-a 被称为命令的参数，所以 uname 命令的作用就像是你在问 Shell：你是谁? OK，再问问 Shell 今夕是何年，输入 date，Shell 会打印当前的日期和时间，请记住作为服务器，系统时间的正确性尤为重要，如果时间不对，有些软件不会按预期工作或者会出现莫名其妙的 Bug。

```
~ 🍎 uname
Darwin
~ 🍎 uname -a
Darwin macpro.local 20.1.0 Darwin Kernel Version 20.1.0: Sat Oct 31 00:07:11 PDT 2020; root:xnu-7195.50.7~2/RELEASE_X86_64 x86_64
~ 🍎 date
2020年12月 9日 星期三 15时36分56秒 CST
```

想知道系统运行多久了? 没问题，输入 uptime 后会显示系统运行时间以及当前的负荷情况，如果你感觉到系统很慢很卡的时候，也可以输入 uptime 命令确认一下，看看是不是系统负荷太高了。第一次和 Shell 交互的感觉是不是就像和一个人对话?

```
~ 🍎 uptime
15:36  up 15 days, 43 mins, 5 users, load averages: 2.86 3.28 3.16
```

如果觉得屏幕信息显示太多了，可以输入 clear 清屏，不想玩了，输入 exit 退出当前 Shell 终端，到此你已经成功的开启了 Shell 的大门，不管是多么复杂的工作，以后都可以在 Shell 终端里输入命令完成，最重要的是你亲手去试一试，下面是视频版。
