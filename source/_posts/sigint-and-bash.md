---
title: 关于Ctrl+C，Bash，SIGINT，进程组的一些踩坑经验
category: 程序员的自我修养
date: 2022-06-07
---

Shell脚本一直被我视为上古的黑科技，从来都是是敬而远之。然后最近工作中遇到一个进程无法按照预期退出的问题，着实让我产生了困扰，不得不静下心来再补习一些基础知识。
问题是这样的：我的一个Python程序，在运行过程中，使用 Subprocess 启动了一个 Shell 脚本进程，并且获取到了进程的 pid。并且在适当的时候，我需要给进程发送 SIGINT 信号，触发进程正常退出。将这个Shell脚本简化，并且以 ping 程序代替实际调用的可执行程序之后，大概如下所示：

```shell
#!/bin/bash

function onSigint() {
echo "sigint captured"
}

function go() {
trap "onSigint" SIGINT

ping 127.0.0.1 &
pid=$!

echo "! Waiting for ping process to exit"
wait $pid
echo "! Ping process exited or shell interrupted"

if ps -p $pid >/dev/null; then
	echo "! process ${pid} still alive, try kill it"
	kill -2 $pid
fi
wait $pid
echo "! Ping process exited or shell interrupted"
}

go | tee -i /tmp/test_log.txt
```

这个脚本我们暂时取名为test.sh，我遇到的问题就是，这个程序在Terminal运行的过程中，Ctrl + C 可以触发进程正常的退出，但是我的Python程序中，给我启动的子进程发 SIGINT却没有任何反应。为了解释其中的原理，需要了解一些容易被忽略的基础原理。

## 1. 当你按 Ctrl + C 时，究竟发生了什么
在一般的认知中，Ctrl + C 等同于给终端的前台进程发送 SIGINT 信号，但是精确来说，其实是给**前台进程组（Foreground Process Group)** 中的所有进程发送 SIGINT 信号 。而前台进程组可以理解为在当前Termnial可以接受你键盘交互的进程，以及由它启动的所有子进程。

下面这张图很好的解释了前台进程组以及一些额外的概念（Session，Session Leader），图片出自[Killing a process and all of its descendants](http://morningcoffee.io/killing-a-process-and-all-of-its-descendants.html)，

![Session, Foreground process group](http://morningcoffee.io/images/killing-a-process-and-all-of-its-descendants/sessions.png)

通过`ps j -A`可以很清楚的看到每个进程的PID，PPID（父进程ID），PGID（进程组ID），SID（Session ID），在这里，我们在一个终端运行 `bash ./test.sh` 之后，看一下进程信息：
```shell
$ ps -j A
PPID   PID  PGID   SID TTY      TPGID STAT   UID   TIME COMMAND
 2596  2668  2668  2596 pts/1     2668 S+       0   0:00 bash ./test.sh
 2668  2669  2668  2596 pts/1     2668 S+       0   0:00 bash ./test.sh
 2668  2670  2668  2596 pts/1     2668 S+       0   0:00 tee -i /tmp/test_log.txt
 2669  2671  2668  2596 pts/1     2668 S+       0   0:00 ping 127.0.0.1
```

可以看出这一个脚本启动了四个进程，至于为什么是四个进程，而且其中包含两个同名进程之后再说。这里面 STAT 一列中的 "+" 号表示该进程属于前台进程组，TPGID则是前台进程组的 ID。**可以看出，虽然我们在脚本中把 ping 程序 通过 "&" 符号放到了后台，但它依然属于前台进程组**。在键盘按下 Ctrl + C 之后，上面所有的进程均会收到 SIGINT 信号。


## 2. Bash进程接收到 SIGINT 信号时，会怎样？
这篇文章[Why Bash is like that: Signal propagation](https://www.vidarholen.net/contents/blog/?p=34)，给出了解释

> When the shell is interrupted, it will wait for the running command to exit. If this child’s status indicates it exited abnormally due to that signal, the shell cleans up, removes its signal handler, and kills itself again to trigger the OS default action (abnormal exit). Alternatively, it runs the script’s signal handler as set with `trap`, and continues.
> 
> If the shell is interrupted and the child’s status says it exited normally, then Bash assumes the child handled the signal and did something useful, so it continues executing. **Ping and top both trap SIGINT and exit normally**, which is why Ctrl-C doesn’t kill the loop when calling them.

总结下来，Bash进程对于SIGINT信号的处理有几点特性：
1. Bash进程一定会等待当前程序运行结束后，再处理 SIGINT 信号。如果当前程序返回值为1，则直接退出脚本。如果返回值为 0，则继续执行脚本。
2. 如果Bash脚本中使用了 trap 命令，捕捉了信号，那么Bash会在当前程序运行结束之后，先运行trap指令中指定的函数指令，然后再继续运行接下来的指令，无论当前程序返回值是什么。
3. 如果当前Bash脚本执行的是 wait 命令，该命令会被直接打断，此时如果没有设置信号捕捉的话，脚本会直接退出。（wait可以理解为Bash程序的内部指令，可以被打断）

由于Bash的这个特性，会产生一些看起来很奇怪的现象。比如说在一个循环中调用 sleep 命令。通过Ctrl + C 可以直接退出脚本，而如果循环中调用的是 ping 命令，则 Ctrl + C 会让脚本进入下一个循环。这是因为按下 Ctrl + C 以后，ping 和 sleep 都会接收到 SIGINT 信号，并立即退出，但 sleep 的返回值不为0，而ping的返回值为0。

```shell
$ sleep 10
^C
$ echo $?
130
$ ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.018 ms
^C
--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.018/0.018/0.018/0.000 ms
$ echo $?
0
```

回到我们上面的 `test.sh` 脚本，其中对信号进行了捕捉，并且使用了两次wait，在两次wait中间还再次给目标程序发送信号。那么如果在脚本运行过程中按下 Ctrl + C，则会发生：
- ping程序退出
- wait被打断，触发onSigint函数，然后继续执行接下来的逻辑
- 由于ping程序已经退出，找不到对应pid，不会再次执行 kill 命令
- 由于ping程序已经退出，第二个wait也直接完成，脚本结束

如果不是Ctrl + C，而是只给执行shell脚本的bash进程发送 SIGINT，则会发生：
- wait 被打断，触发 onSigint函数，然后继续执行接下来的逻辑
- 由于 ping 程序依然活着，if判断下面的 kill 命令会被执行，给 ping 进程发送SIGINT 信号
- 第二个 wait 在 ping 程序退出之后结束，脚本结束

所以按照`test.sh`的写法，似乎只给 Bash进程发 SIGINT 信号，也是可以实现正常退出的，那么为什么在我最初提到的问题中，会失败呢。这与脚本最后使用的管道命令有关

## 3. 在Bash脚本使用函数加管道会发生什么
上面第一点提到了存在两个同名进程的问题，我们使用`ptree -p` 指令看一下进程之间的层级关系：

```shell

$ ptree -p

systemd(1)─sshd(668)─┬─sshd(2576)───bash(2596)───bash(2668)─┬─bash(2669)───ping(2671)
                                                            └─tee(2670)
```

可以看到由于`go | tee -i /tmp/test_log.txt` 的存在，bash 额外启动了一个子线程（可能是直接fork的方式，进程名完全相同）进行函数内 Shell脚本的执行，并将管道接入 tee 的子线程中。此时执行脚本的顶层 Bash 进程实际上没有 Trap 任何 信号，只会按照之前提到的默认规则进行响应，即等待当前命令执行完毕。

而我通过Python Subprocess 调用 `test.sh` 拿到的，正是顶层 Bash 进程的 PID，对它发送SIGINT信号，自然也得不到任何响应。

## 解决问题
大部分网上的问题解答会提供这样的方法，即通过 在 pid前面加 '-' 减号，来达到给整个进程组发送信号的目的。（其中的 "--" 两个短横线是告诉shell不要把后面的 "-" 误认为参数标志）
```shell
kill -s SIGINT -- -$gpid
```

但在我的问题中，这样做有点棘手，因为 shell 脚本是在 python 程序的子线程里执行的，事实上，bash进程和python进程属于同一个进程组。我当然不想让 python 进程也收到信号。

我最后解决问题的方法很简单，不过也非常局限于我当前的问题，即通过 `pkill` 发送信号给我启动的`bash test.sh`进程的直接子线程。
```python
# start bash process
p = subprocess.Popen(["bash", "test.sh"])

# send SIGINT to direct children of the bash process
subprocess.call("pkill --signal SIGINT -P {}".format(p.pid), shell=True)

# wait process exit
p.communicate()
```

要注意的是，此处在python中启动 `test.sh` 必须用列表命令的方式，而不能用字符串加参数`shell=True`，否则 python 又会再起一个 线程作为`bash test.sh` 的父线程，这样我们拿到的 pid 的会又多了一层进程层级关系，这个方法又不work了。

其实，改动`./test.sh`让它能够独立妥善处理 SIGINT 才是最优雅和最不容易埋坑的做法。

## 额外提一点，关于孤儿进程
问：如果父进程退出了，而子进程还正常运行会怎么样呢？
答：被所有进程之母收养（systemd 或 init）

参考4. [Killing a process and all of its descendants](http://morningcoffee.io/killing-a-process-and-all-of-its-descendants.html)


## 参考资料
1. [Why Bash is like that: Signal propagation](https://www.vidarholen.net/contents/blog/?p=34)
2. [Why is SIGINT not propagated to child process when sent to its parent process?](https://unix.stackexchange.com/questions/149741/why-is-sigint-not-propagated-to-child-process-when-sent-to-its-parent-process)
3. [SIGINT Propagation Between Parent and Child Processes](https://www.baeldung.com/linux/signal-propagation)
4. [Killing a process and all of its descendants](http://morningcoffee.io/killing-a-process-and-all-of-its-descendants.html)
5. [Process groups and sessions](https://www.andy-pearce.com/blog/posts/2013/Aug/process-groups-and-sessions/)