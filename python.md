# 1. 指定非默认路径的module

```python
sys.path.append("/root/scapy-2.4.0")
from scapy.all import *
```

&nbsp;


# 2. 多进程

```python
from multiprocessing import Process

def f(name):
    print 'hello', name

if __name__ == '__main__':
    p = Process(target=f, args=('bob',))
    p.start()
    p.join()
```