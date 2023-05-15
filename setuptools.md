
# 1. Simple example

## 1.1 Setup a demo

```sh
$ tree ./
./
├── myapp
│   └── __init__.py
└── setup.py

1 directory, 2 files
```

myapp/\_\_init\_\_.py:
```python
def test():
    print("Hello World!")


if __name__ == '__main__':
    test()
```

setup.py:
```python
from setuptools import setup, find_packages

setup (
    name = 'HelloWorld',
    version = '0.1',
    packages = find_packages(),
    author = 'xxx',
    url = 'None',
    author_email = 'None'
)
```

## 1.2 Check the validity of setup.py
```sh
$ python setup.py check
running check
```

## 1.3 Package
```sh
$ python setup.py bdist_egg
...
```

最终的文件有：
```sh
$ tree .
.
├── build
│   ├── bdist.linux-x86_64
│   └── lib.linux-x86_64-2.7
│       └── myapp
│           └── __init__.py
├── dist
│   └── HelloWorld-0.1-py2.7.egg
├── HelloWorld.egg-info
│   ├── dependency_links.txt
│   ├── PKG-INFO
│   ├── SOURCES.txt
│   └── top_level.txt
├── myapp
│   └── __init__.py
└── setup.py

7 directories, 8 files
```
在dist中生成的是egg包，.egg文件其实是一个zip包。

## 1.4 Install
解压egg包后，安装该包：
```sh
$ python setup.py install
```

&nbsp;


# 2. Console scripts used in entry points
console_scripts允许Python的functions(不是文件)直接被注册成一个命令行工具。

增加myapp/command_line.py：
```python
import myapp

def main():
    myapp.test()
```

修改setup.py：
```python
from setuptools import setup, find_packages

setup (
    name = 'HelloWorld',
    version = '0.1',
    packages = find_packages(),
    author = 'xxx',
    url = 'None',
    author_email = 'None',
    entry_points = {
        'console_scripts' : ['mytest=myapp.command_line:main'],
    }
)
```

安装dist/HelloWorld-0.1-py2.7.egg：
```sh
$ sudo easy_install ./HelloWorld-0.1-py2.7.egg
Processing HelloWorld-0.1-py2.7.egg
Copying HelloWorld-0.1-py2.7.egg to /usr/local/lib/python2.7/dist-packages
Adding HelloWorld 0.1 to easy-install.pth file
Installing mytest script to /usr/local/bin

Installed /usr/local/lib/python2.7/dist-packages/HelloWorld-0.1-py2.7.egg
Processing dependencies for HelloWorld==0.1
Finished processing dependencies for HelloWorld==0.1
```

命令行可以直接执行mytest：
```sh
$ file /usr/local/bin/mytest
/usr/local/bin/mytest: Python script, ASCII text executable
```

Reference:
<https://setuptools.pypa.io/en/latest/userguide/entry_point.html#>
