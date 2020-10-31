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



