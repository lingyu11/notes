# Lectures

## 资料
* [CSDIY](https://csdiy.wiki/%E7%BC%96%E7%A8%8B%E5%85%A5%E9%97%A8/Python/CS61A/#_1)
* [课程网站(英文版)](https://cs61a.org/), [课程网站(中文版)](https://chillyhigh.github.io/CS61A-CN/)
* [课程视频](https://www.bilibili.com/video/BV1s3411G7yM/?vd_source=2a33d03ec3e67e46971208a7faa0dcda)
* [课本(英文版)](https://www.composingprograms.com/), [课本(中文版)](https://composingprograms.netlify.app/1/1)
* [Python Tutor Visualizer](https://pythontutor.com/cp/composingprograms.html#mode=edit)

## Higher-Order Functions

1. lambda 表达式

    ```python
    def search(f):
        x = 0
        while not f(x):
            x += 1
        return x

    def square(x):
        return x * x

    def inverse(f):
        """返回函数 f 的反函数: 它接收一个函数f作为参数，并返回一个新的函数（lambda表达式）。
        这个新函数对于给定的输入y，会寻找一个x使得f(x)等于y"""
        return lambda y: search(lambda x: f(x) == y)

    sqrt = inverse(square)
    print(sqrt(16))  # 输出 4
    ```

2. 以下函数在参数为-4时报错的原因是：`return if_(x > 0, sqrt(x), 0.0)`是一个调用表达式，在调用它之前，对其三个参数都进行了评估，因此会调用`sqrt(-4)`，而`sqrt(-4)`会报错。

    ```python
    def if_(c, t, f):
        if c:
            return t
        else:
            return f

    from math import sqrt

    def real_sqrt(x):
        return if_(x > 0, sqrt(x), 0.0)

    real_sqrt(4)   # 输出 2.0
    real_sqrt(-4)  # 报错：math domain error
    ```