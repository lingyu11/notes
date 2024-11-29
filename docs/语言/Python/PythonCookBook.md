# Python Cook Book

## 资料
* 在线教材：[python3-cookbook](https://python3-cookbook.readthedocs.io/zh-cn/latest/index.html).

## 解压可迭代对象赋值给多个变量
* 星号表达式可以用来解决这个问题。比如，你在学习一门课程，在学期末的时候， 你想统计下家庭作业的平均成绩，但是排除掉第一个和最后一个分数：

```python
def drop_first_last(grades):
    first, *middle, last = grades
    return avg(middle)
```

## 单下划线和双下划线
* Python程序员不去依赖语言特性去封装数据，而是通过遵循一定的属性和方法命名规约来达到这个效果。 
* 第一个约定是任何以单下划线_开头的名字都应该是内部实现。
* 使用双下划线开始会导致访问名称变成其他形式。
*  大多数而言，你应该让你的非公共名称以单下划线开头。但是，如果你清楚你的代码会涉及到子类， 并且有些内部属性应该在子类中隐藏起来，那么才考虑使用双下划线方案。
* [Reference](https://python3-cookbook.readthedocs.io/zh-cn/latest/c08/p05_encapsulating_names_in_class.html)

## 装饰器
* 一个装饰器就是一个函数，它接受一个函数作为参数并返回一个新的函数。
```python
import time
from functools import wraps

def timethis(func): # timethis是个装饰器
    '''
    Decorator that reports the execution time.
    '''
    @wraps(func) # 作用是将被装饰函数的元数据（如函数名、文档字符串等）复制到
    # 装饰器内部的函数 wrapper 中，这样可以避免在使用装饰器后丢失原函数的元数据。
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(func.__name__, end-start)
        return result
    return wrapper
```
这两者等价：
```python
@timethis
def countdown(n):
    while n > 0:
        n -= 1
```
```python
def countdown(n):
    while n > 0:
        n -= 1
countdown = timethis(countdown)
```
用法：
```python
>>> countdown(100000)
countdown 0.008917808532714844
```


* `@property` 是Python中的一种装饰器语法，用于定义属性访问器。它允许你**将类的方法转换为属性**，从而可以通过点号（.）操作符来访问这些方法，而不需要调用它们。

```python
class Person:
    def __init__(self, first_name):
        self._first_name = first_name

    # Getter function
    @property
    def first_name(self):
        return self._first_name # example: a.first_name, note: no () needed

    # Setter function
    @first_name.setter
    def first_name(self, value):
        if not isinstance(value, str):
            raise TypeError('Expected a string')
        self._first_name = value

    # Deleter function (optional)
    @first_name.deleter
    def first_name(self):
        raise AttributeError("Can't delete attribute")
```
