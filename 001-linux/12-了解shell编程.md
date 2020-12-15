终于说到了 shell 编程，为了提高工作效率，经常我们会运行 shell 脚本来自动化的完成一些工作，shell 编程很强大但其实不需要深入学习(除非你的工作就是天天写 shell 脚本)，很多年以前就有人用 shell 编写了[俄罗斯方块游戏](https://www.unix.com/shell-programming-and-scripting/174525-tetris-game-based-shell-script-new-algorithm.html)，而且还在[不断更新](https://www.omgubuntu.co.uk/2019/06/play-tetris-terminal-linux)，先看最简单的 hello world 脚本:

```bash
#!/bin/bash 
#特殊注释，表示使用 bash 这种 shell 来运行这个脚本

hi="hello world"
echo $hi #打印字符串
now=`date` #记录当前时间
echo $now #打印变量

if [ $? -eq 0 ]; then
  echo 执行成功了
fi
```

所谓 shell 脚本不过是一堆命令的组合而已，但是编写过程中经常需要有逻辑性判断，就需要了解隐含变量、判断、循环、分支等知识，为了让脚本维护性更好，还需要了解函数知识，shell 脚本可以很简单，也可以很复杂，先学习一个大概印象即可。

上面的脚本使用了变量和 if 条件判断，变量可以显示的赋值，就像 hi，也可以来自其他命令的输入，就像now，如果不喜欢`date`的写法，可以写成now=$(date)，注意等号两边没有空格。if 的条件判断引用了隐含变量 $? 使用 -eq 运算符判断上个命令退出码是否为0，if 可以支持的判断非常多，比如判断文件是否、数值是否相等、文件的属性、字符创的长度、变量是否相等、逻辑表达式是否成立等等。在编写脚本的时候，经常需要根据目标环境使用不同的命令或者参数，下面的脚本就是判断目标环境的 linux 系统类型。

```bash
#!/bin/bash


os=`uname -s`

if [ $os = "Linux" ] ; then
    echo "Linux"
elif [ $os = "FreeBSD" ] ; then
    echo "FreeBSD"
elif [ $os = "Solaris" ] ; then
    echo "Solaris"
else
    echo "What?"
fi
```

由于这个函数太过于常用，我们决定把它写到单独的文件中，以便可以重复使用，把公用的文件叫 lib.sh，并且把它封装成一个函数。

```bash
#!/bin/bash

getos() {
	os=`uname -s`

	if [ $os = "Linux" ] ; then
		echo "Linux"
	elif [ $os = "FreeBSD" ] ; then
		echo "FreeBSD"
	elif [ $os = "Solaris" ] ; then
		echo "Solaris"
	else
		echo "What?"
	fi
}
```

写一个测试文件 callos.sh 来调用它。

```bash
#!/bin/bash

source ./lib.sh // 执行另外一个脚本文件
getos 
```

你可能听过说函数有参数，但是 shell 脚本中函数不能定义参数，那么如何传递参数呢? 就需要了解一些隐含变量了。比如 $1 是第一个参数，$2 是第二个参数，${10} 是第十个参数，$# 是参数的个数，$* 是传递的所有参数，$$ 是脚本运行的当前进程 PID 等等。函数返回值是函数体内部最后一条指令的 $?，但是也可以使用 return 明确返回一个数字值，是的，只能是数字，如果要返回其他类型的可以在函数中把结果赋值给全局变量，其他地方访问全局变量获取结果。

在 lib.sh 文件中增加一个打印函数参数的函数。

```bash
printargs() {
        for arg in $*; do
                echo $arg
        done
}
```

用了 for 循环来遍历 $*，写一个脚本来调用它。

```bash
#!/bin/bash

source ./lib.sh
printargs 1 one 2 two
```

你应该对 shell 脚本有了基本的认识了，记住测试的时候别忘了给你的脚本加上 x 可以执行权限。shell 脚本可以通天，但是对于网络上下载的 shell 脚本，你要小心，可能会恶意损害你的系统，不要随意以管理员身份来运行你不清楚的 shell 脚本。
