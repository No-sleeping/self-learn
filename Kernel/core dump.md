### 1、什么是 Core Dump

Core Dump 又叫核心转储。在程序运行过程中发生异常时，将其内存数据保存到文件中，这个过程叫做 Core Dump。

### 2、Core Dump 的作用

在开发过程中，难免会遇到程序运行过程中异常退出的情况，这时候想要定位哪里出了问题，仅仅依靠程序自身的信息打印（日志记录）往往是不够的，这个时候就需要 Core Dump 文件来帮忙了。

一个完整的 Core Dump 文件实际上相当于恢复了异常现场，利用 Core Dump 文件，可以查看到程序异常时的所有信息，变量值、栈信息、内存数据，程序异常时的运行位置（甚至记录代码行号）等等，定位所需要的一切信息都可以从 Core Dump文件获取到，能够非常有效的提高定位效率。

### 3、如何生成 Core Dump

#### 3.1 Core Dump 文件生成开关

Core Dump 文件的生成是由Core文件大小限制，Linux中默认的Core文件大小设置为零，也就是不生成 Core Dump 文件，可以使用ulimit -c命令来查看当前的Core文件大小限制。

要生成 Core Dump 文件，只需要执行下面的命令设置Core文件的大小即可（其中filesize参数的单位为KByte）：`ulimit -c filesize`
如果要设置Core文件大小无限制（即把程序完整的运行内存都保存下来），则执行如下命令：`ulimit -c unlimited`

4、core dump 文件生成

Linux 中信号是一种异步事件处理的机制，每种信号对应有其默认的操作，你可以在 这里 查看 Linux 系统提供的信号以及默认处理。默认操作主要包括忽略该信号（Ingore）、暂停进程（Stop）、终止进程（Terminate）、终止并发生core dump（core）等。

那么，以下列出几种信号，它们在发生时会产生 core dump：

![](https://img-blog.csdnimg.cn/46d71a240fa94923a43047b369cf34f1.png)

当然不仅限于上面的几种信号。这就是为什么我们使用 Ctrl+z 来挂起一个进程或者 Ctrl+C 结束一个进程均不会产生 core dump，因为前者会向进程发出 SIGTSTP 信号，该信号的默认操作为暂停进程（Stop Process）；后者会向进程发出SIGINT 信号，该信号默认操作为终止进程（Terminate Process）。

同样上面提到的 kill -9 命令会发出 SIGKILL 命令，该命令默认为终止进程。

而如果我们使用 Ctrl+\ 来终止一个进程，会向进程发出 SIGQUIT 信号，默认是会产生 core dump 的。还有其它情景会产生 core dump， 如**：程序调用 abort() 函数、访存错误、非法指令**等等。

<br/>

参考：

https://blog.csdn.net/hongge_smile/article/details/125028302
