# python 多进程编程

> 参考：https://juejin.im/post/6844903613353951240 and https://juejin.im/post/6844903613425270797

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

