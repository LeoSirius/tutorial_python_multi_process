# python 多进程编程

> 参考：https://juejin.im/post/6844903613353951240 and https://juejin.im/post/6844903613425270797

第一部分：进程基础

- [生成子进程](#生成子进程)
- [生成多个子进程](#生成多个子进程)
- [杀死子进程](#杀死子进程)
- [僵尸进程](#僵尸进程)
- [收割子进程](#收割子进程)
- [捕获信号](#捕获信号)

第二部分：进程间通信

- [多进程并行计算实例](#多进程并行计算实例)

## 生成子进程

创建一个进程用`os.fork()`。返回值`pid`有三种情况

- 在父进程中，pid为子进程的id
- 在子进程中，pid为0
- 出错，pid小于0

这里封装一个函数来创建子进程

```py
# client.py
import os

def create_child():
    pid = os.fork()
    if pid > 0:
        print('in parent process')
    elif pid == 0:
        print('in child process')
    else:
        raise Exception
```

## 生成多个子进程

```py
# client.py

import os

def create_child(i):
    pid = os.fork()
    if pid > 0:
        print('in parent process')
    elif pid == 0:
        print(f'in child process {i}')
    else:
        raise Exception

    return pid

for i in range(10):
    pid = create_child(i)

    # 如果是子进程，直接退出。不然会很快生成很多进程
    if pid == 0:
        break
```

运行输出

```
leo@192 test $ python3 tmp.py 
in parent process
in child process 0
in parent process
in child process 1
in parent process
in child process 2
in parent process
in child process 3
in parent process
in child process 4
in parent process
in parent process
in child process 5
in parent process
in child process 6
in child process 7
in parent process
in child process 8
in parent process
in child process 9
```

## 杀死子进程

`os.kill(pid, sig_num)`可以向子进程pid发送信号，常用的信号有

- SIGKILL，暴力杀死，相当于`kill -9`
- SIGTERM，通知对方退出
- SIGINT，相当于键盘的`ctrl+c`

```py
# kill.py

import os
import time
import signal

def create_child():
    pid = os.fork()
    if pid < 0:
        raise Exception
    return pid

pid = create_child()
if pid == 0:
    while True:    # 在子进程中循环打印
        print('in child process')
        time.sleep(1)

else:
    print('in parent process')
    time.sleep(5)                    # 父进程休眠5s再杀死子进程
    os.kill(pid, signal.SIGKILL)
    time.sleep(5)                    # 父进程继续休眠5s观察子进程是否还有输出
```

运行输出

```
leo@192 test $ python3 tmp.py 
in parent process
in child process
# 等1s
in child process
# 等1s
in child process
# 等1s
in child process
# 等1s
in child process
# 等1s
in child process
# 最后等待5秒后退出
```

## 僵尸进程

而僵尸进程就是指：一个进程执行了exit系统调用退出，而其父进程并没有为它收尸(调用wait或waitpid来获得它的结束状态)的进程。

在上面的例子中，os.kill执行完之后，我们通过`ps -ef|grep python`快速观察进程的状态，可以发现子进程有一个奇怪的显示`<defunct>`。待父进程终止后，子进程也一块消失了。

在`kill.py`中加上这个命令看看输出

```py
# kill.py
import os
import time
import signal

def create_child():
    pid = os.fork()
    if pid < 0:
        raise Exception
    return pid

pid = create_child()
if pid == 0:
    while True:    # 在子进程中循环打印
        print('in child process')
        time.sleep(1)

else:
    print('in parent process')
    time.sleep(5)                    # 父进程休眠5s再杀死子进程
    os.kill(pid, signal.SIGKILL)
    print(os.system('ps -ef|grep python'))
    time.sleep(5)                    # 父进程继续休眠5s观察子进程是否还有输出
```

输出

```
(base) root slt (master) # python tmp.py 
in parent process
in child process
in child process
in child process
in child process
in child process
root     13080 20396  0 18:27 pts/1    00:00:00 python tmp.py
root     13081 13080  0 18:27 pts/1    00:00:00 [python] <defunct>
```

## 收割子进程

`waitpid(pid, options)`父进程中调用这个函数可以收割子进程。这个函数的返回值是一个tuple。第一个元素是子进程的id。第二个元素在不同的操作系统上含义是不同的。在Unix上，它通常的value是一个16位的整数值，前8位表示进程的退出状态，后8位表示导致进程退出的信号的整数值。

```py
import os
import time
import signal


def create_child():
    pid = os.fork()
    if pid < 0:
        raise Exception
    return pid


pid = create_child()
if pid == 0:
    while True:
        print('in child process')
        time.sleep(1)

else:
    print('in parent process')
    time.sleep(5)
    os.kill(pid, signal.SIGTERM)
    ret = os.waitpid(pid, 0)
    print(f'ret = {ret}')
    time.sleep(5)
```

打印输出

```
leo@192 test $ python3 tmp.py 
in parent process
in child process
in child process
in child process
in child process
in child process
in child process
ret = (13191, 15)
```

## 捕获信号

在默认情况下，子进程收到`SIGTERM`信号就会自己退出。我们也可以自定义信号的处理函数

我们让子进程忽略`SIGTERM`信号，看看会有什么效果

```py
import os
import time
import signal


def create_child():
    pid = os.fork()
    if pid < 0:
        raise Exception
    return pid


pid = create_child()

if pid == 0:
    # 这里设置子进程的信号处理，SIG_IGN会忽略SIGTERM信号
    signal.signal(signal.SIGTERM, signal.SIG_IGN)
    while True:
        print('in child process')
        time.sleep(1)

else:
    print('in parent process')
    time.sleep(5)
    os.kill(pid, signal.SIGTERM)    # 向子进程发退出信号
    print('SIGTERM send...')
    time.sleep(5)
    os.kill(pid, signal.SIGKILL)    # 向子进程发SIGKILL信号
    print('SIGKILL send...')
    time.sleep(5)
```

可以看到，在父进程发了`SIGTERM`后，子进程忽略的这个信号，并继续运行。直到父进程发`SIGKILL`信号

```
leo@192 test $ python3 tmp.py 
in parent process
in child process
in child process
in child process
in child process
in child process
in child process
SIGTERM send...
in child process
in child process
in child process
in child process
in child process
SIGKILL send...
```

`signal(int signum, sighandler_t handler)`函数的第二个参数处理预定义的，也可以自己实现

信号处理函数有两个参数，第一个sig_num表示被捕获信号的整数值，第二个frame不太好理解，一般也很少用。它表示被信号打断时，Python的运行的栈帧对象信息。读者可以不必深度理解。

```py
import os
import time
import signal
import sys


def create_child():
    pid = os.fork()
    if pid < 0:
        raise Exception
    return pid

def i_will_die(sig_num, frame):
    print('child will die')
    sys.exit(0)

pid = create_child()
if pid == 0:
    signal.signal(signal.SIGTERM, i_will_die)
    while True:
        print('in child process')
        time.sleep(1)

else:
    print('in parent process')
    time.sleep(5)
    os.kill(pid, signal.SIGTERM)
    time.sleep(5)
```

这次再运行，可以看到自定义handler中的输出

```
leo@192 test $ python3 tmp.py 
in parent process
in child process
in child process
in child process
in child process
in child process
child will die
```

## 多进程并行计算实例

我们用下面的公式来计算圆周率PI

![圆周率公式](https://s1.ax1x.com/2020/11/03/Byg3Hf.png)

首先我们直接在一个进程中进行1亿次的迭代运算

```py
import time
import math

def pi(n):
    s = 0.0
    for i in range(n):
        s += 1.0/(2*i+1)/(2*i+1)
    return math.sqrt(8 * s)

start = time.time()
print(pi(100000000))
end = time.time()

print(f'time: {end - start}')
```

结果是耗时20多秒

```
leo@192 test $ python3 tmp.py 
3.141592647851581
time: 23.279133796691895
```
