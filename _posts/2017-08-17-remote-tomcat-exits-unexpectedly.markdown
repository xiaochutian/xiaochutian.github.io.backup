---
layout:     post
title:      "远程tomcat意外退出"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - tomcat
    - ssh
---

## 0 一篇一句

**请用我喜欢的方式来爱我**    

## 1 概述

#### 1.1 需求
启动远程服务器上的tomcat，并且要求tomcat在启动脚本结束之后继续运行。
```bash
# 启动脚本
ssh -t -t username@remote_host 'sh run_tomcat.sh'
```

```bash
# 远程服务器的 run_tomcat.sh 脚本
$ cat run_tomcat.sh
#!/bin/bash
cd /data/server/tomcat/bin/
./catalina.sh start
tail -f /data/server/tomcat/logs/catalina.out
```

#### 1.2 问题
远程的tomcat，会在启动之后意外退出。（时间点正好是 **启动脚本执行完成的时候**）

## 2 解决办法

#### 2.1 远程脚本使用set -m
启动脚本不需要改。**在远程脚本的开头部分，添加set -m**
```bash
# 修改后的 run_tomcat.sh 脚本
$ cat run_tomcat.sh
#!/bin/bash
set -m
cd /data/server/tomcat/bin/
./catalina.sh start
tail -f /data/server/tomcat/logs/catalina.out
```

#### 2.2 使用nohup
```bash
ssh username@remote_host 'nohup sh ~/run_tomcat.sh `</dev/null` >nohup.out 2>&1 &'
```

#### 2.3 使用screen
Screen命令集合
```bash
screen -S name
screen -r name
screen -wipe
screen -ls
screen -X -S 27171 quit

# Ctrl+a d退出
```

## 3 原理

#### 3.1 简而言之

- 启动脚本在退出的时候，通过sshd传递SIGHUP信号给远程服务器的bash进程
- bash会把SIGHUP信号传递给它的子进程 run_tomcat.sh，并把SIGHUP信号传递给 **子进程相同进程组成员**
- Java进程从父进程 catalina.sh（父进程为 run_tomcat.sh）继承pgid，即Java也属于 run_tomcat.sh进程组成员
- Java进程收到SIGHUP信号后退出

> 注：使用set -m之后，catalina.sh不再使用run_tomcat.sh的进程组，而使用自己的pid作为pgid，catalina.sh进程在执行完退出后，Java进程挂到了init下，Java与run_tomcat.sh进程就完全脱离关系了，bash也不会再向它发送信号。

**『解决办法：让Java进程接收不到由启动脚本产生的SIGHUP信号』**      
详细分析见：[【内存溢出】Tomcat进程意外退出的问题分析](http://ju.outofmemory.cn/entry/67694)

#### 3.2 使用 set -m 的原理

**『让catalina.sh与bash在不同的进程组，Java进程接收不到SIGHUP信号』**

```
-m      Monitor  mode. [...]                     
        Background processes run in a separate process group
        and a line containing their exit status is printed upon their completion.
```

> set +m下启动的后台作业不是实际的『background processes』（后台进程是那些进程组ID不同于终端的）：它们与启动它们的shell共享相同的进程组ID，而不是拥有自己的进程组像正确的后台进程。退出时会将信号发送到与shell相同的进程组中的进程（从而杀死在set +m下启动的后台作业），但不会杀死set -m下启动的进程。

详细分析见：[【共享笔记】有人可以详细解释“set -m”吗？](https://gxnotes.com/article/126205.html)  

## 4 其他

- Linux信号（signal）
- [【IBM developerWorks】Linux 技巧：让进程在后台可靠运行的几种方法](https://www.ibm.com/developerworks/cn/linux/l-cn-nohup/)    
- [【共享笔记】执行远程命令，完全脱离ssh连接](https://gxnotes.com/article/15510.html)

## 5 参考

[【内存溢出】Tomcat进程意外退出的问题分析](http://ju.outofmemory.cn/entry/67694)    
[【共享笔记】执行远程命令，完全脱离ssh连接](https://gxnotes.com/article/15510.html)    
[【CSDN-David Camp】linux screen 命令详解](http://www.cnblogs.com/mchina/archive/2013/01/30/2880680.html)    
[【共享笔记】有人可以详细解释“set -m”吗？](https://gxnotes.com/article/126205.html)    
[【Stack Overflow】Can someone explain in detail what “set -m” does?](https://unix.stackexchange.com/questions/196603/can-someone-explain-in-detail-what-set-m-does)    
[【IBM developerWorks】Linux 技巧：让进程在后台可靠运行的几种方法](https://www.ibm.com/developerworks/cn/linux/l-cn-nohup/)    
