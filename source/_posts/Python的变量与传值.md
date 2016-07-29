---
title: Python的变量与传值
date: 2016-07-28 16:26:18
categories: [Coding]
tags: [Python]
---

Python设计变量时候做了一个很有趣的决定，所有变量的传递皆为引用，出发点无外乎让不同变量共享内存，提高效率。其实引用的定义和用法一直迷惑了不少Python的新手，但如果有C/C++经验的程序员来学习Python，那么对他来说所有变量的本质其实就是指针，其意义再清晰不过了。

<!--more-->

# 可变与不可变

Python把对象分成两类，可变对象与不可变对象。
可变(mutable): `dict`, `list`.
不可变(immutable): `int`, `str`, `float`, `tuple`.

不可变对象值发生改变后会创建并返回一个**全新**的对象。
```python
>>> s = "Hello"
>>> print(id(s))
139641239382144
>>> s = s + ", world!"
>>> print(id(s))
139641239393520
```
字符串`s`指向的内存地址发生了变化，说明操作前后已经<font color=red>不是同一个对象</font>.
![demo](/images/Python的变量与传值/demo.png)


可变对象内容发生改变后依然返回**相同**的对象。
```python
>>> a = []
>>> print(id(a))
139641239842568
>>> a.append(1)
>>> print(id(a))
139641239842568
```
列表`a`指向的内存地址没有变化，说明操作前后依然是同一个对象。


# 传值

Python函数传递的所有变量都是引用,在这种情况下,如果不可变对象的值发生的变化,返回了生成的新对象,原对象的值自然不会发生变化.

试用`swap()`函数来交换不可变对象`str`的值.
```python
def swap(a, b):
    a, b = b, a
    print("The value of (a, b) in test() is ",a, b)

def main():
    a = "a"
    b = "b"
    swap(a, b)
    print("The value of (a, b) in main() is ",a, b)

if __name__ == '__main__':
    main()
```

结果并没有改变原来`a`, `b`的值,因为swap()中的`a`, `b`指向的是生成的新对象.
```
➜  python ./my.py 
The value of (a, b) in test() is  b a
The value of (a, b) in main() is  a b
```
python传值中需要注意的还有一个地方, 默认参数的值只有在函数被加载时候会被计算一次, 因此多次调用函数时候, 默认参数的值可能会发生改变.
```python
def test(val, l=[]):
    l.append(val)
    return l

def main():
    print(test(1))
    print(test(2))
    print(test(3))

if __name__ == '__main__':
    main()
```
结果如下
```
➜  python ./my.py
[1]
[1, 2]
[1, 2, 3]
```
python这个传值的特点大多时候会帮倒忙, 函数的行为依赖于<font color=red>__过去的调用结果__</font>, 这给debug造成了大麻烦, 因此定义默认参数的时候我们会选择设置为_不可变对象_来确保函数行为的前后一致性.

我们根据这点修改`test()`函数
```python
def test(val, l=None):
    if l is None:
        l = []
    l.append(val)
    return l
```
结果如下, 函数行为符合我们的心里预期.
```
➜  python ./my.py
[1]
[2]
[3]

```
