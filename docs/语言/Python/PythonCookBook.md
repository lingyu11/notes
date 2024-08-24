# PythonCookBook

在线教材：[python3-cookbook](https://python3-cookbook.readthedocs.io/zh-cn/latest/index.html).

## 解压可迭代对象赋值给多个变量
* 星号表达式可以用来解决这个问题。比如，你在学习一门课程，在学期末的时候， 你想统计下家庭作业的平均成绩，但是排除掉第一个和最后一个分数：
```python
def drop_first_last(grades):
    first, *middle, last = grades
    return avg(middle)
```
