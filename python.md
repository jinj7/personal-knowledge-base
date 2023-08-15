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

&nbsp;


# 3. List Comprehensions vs Generator Expressions

## 3.1 What is List Comprehension?
It is an elegant way of defining and creating a list. List Comprehension allows us to create a list using for loop with lesser code. What normally takes 3-4 lines of code, can be compressed into just a single line.

```python
# List Comprehension
list_comprehension = [i for i in range(11) if i % 2 == 0]
  
print(list_comprehension)
```

Output:
```text
0 2 4 6 8 10
```

## 3.2 What are Generator Expressions?
Generator Expressions are somewhat similar to list comprehensions, but the former doesn’t construct list object. Instead of creating a list and keeping the whole sequence in the memory, the generator generates the next element in demand.

When a normal function with a return statement is called, it terminates whenever it gets a return statement. But a function with a yield statement saves the state of the function and can be picked up from the same state, next time the function is called.
The Generator Expression allows us to create a generator without the yield keyword.

Syntax Difference: Parenthesis are used in place of square brackets.

```python
# Generator Expression
generator_expression = (i for i in range(11) if i % 2 == 0)
  
print(generator_expression)
```

Output:
```text
<generator object  at 0x000001452B1EEC50>
```

## 3.3 References

[Python List Comprehensions vs Generator Expressions](https://www.geeksforgeeks.org/python-list-comprehensions-vs-generator-expressions/)

[When to Use a List Comprehension in Python](https://realpython.com/list-comprehension-python/)