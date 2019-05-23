---
layout: post
title:  "super in python"
date:   2018-04-04 20:00:05 +0800
categories: python
---

# 问题描述

Java 只允许单继承，创建类很少出现某些奇怪现象，但是 Python 支持多继承 不熟悉 MRO 有可能导致类无法被创建？不相信请尝试以下代码：

```python
O = object

class X(O): pass

class Y(O): pass

class A(X, Y):  pass

class B(Y, X):  pass

class C(A, B):  pass
```

具体原因设计到 MRO 所使用的 C3 算法，笔者有在下面展开分析。

## super 在 Python2 与 Python3 之间的区别

以下代码在 2 和 3 都能够正常运行

```python
class Child(Base):
    def __init__(self):
        super(Child, self).__init__()
```

以下代码只能在 3 运行

```python
class Child(Base):
    def __init__(self):
        super().__init__()
```

## `super().__init__()` 与 `Base.__init__(self)` 的区别

思考以下以下两个代码片段可能产生的效果有什么区别：

```python
# 1
class Child(Base):
    def __init__(self):
        super(Child, self).__init__()
```

```python
# 2
class Child(Base):
    def __init(self):
        Base.__init__(self)
```

**一定要使用代码片段 1，而不应该使用代码片段 2**

- `super(Child, self)` 可以减少硬编码为 `Base`

如果 Python 的解析器能够帮助我们做的事情，我们为什么一定要硬编码？如果未来 Child 的父类改变了，忘了改 `Base.__init__(self)` 那就可能产生灾难。Python 是脚本语言并没有经过完整的编译，上述错误只有在运行时才能够被发现。

- `super()` 可以实现多继承，造成可能像 C++ 一样出现基类重复的情况，C++ 的解决方案是虚基类，那么 Python 呢？ 

Python 支持多继承，假设 `class Child(Base1, Base2)`，那么是不是手动一个一个地调用父类的 `__init__` 方法，如以下丑陋且易错的代码：


```python
class Child(Base1, Base2):
    def __init__(self):
        Base1.__init__(self)
        Base2.__init__(self)
```

> 继承中一定要使用 `super`

如果你在生产环境采用了类似的代码，那么 code review 的时候很可能被公开批评，特别是在多继承的结构变得复杂以后，尤其容易出错。具体分析看下面的 MRO 介绍。

# 正文

## MRO

`super()` 是根据 MRO(Method Resolution Order) 计算的，而 Python 的 MRO 采用了 C3 算法。

### C3 算法

有以下的类结构：

```python
O = object
class F(O): pass
class E(O): pass
class D(O): pass
class C(D,F): pass
class B(D,E): pass
class A(B,C): pass
```

1. 设 L[cls] 为类 cls 到其根父类的路径
2. 设 merge(P1, P2, P3) 操作是从 P1...P3 寻找元素 x，其中符合 x 要么不在 P 中，要么是 P 的第一个元素，如：

```
merge(abc, ac, co)
= a + merge(bc, c, co)
= ab + merge(c, c, co)
= abc + merge(o)
= abco
```

```
L[O] = O
L[F] = FO
L[E] = EO
L[D] = DO
```

上述三个我想读者都不会有异议。下面着重分析 C/B/A：

```
L[C] = C + merge(L[D], L[F], DF)
= C + merge(DO, FO, DF)
= CD + merge(O, FO, F)
= CDF + merge(O)
= CDFO

L[B] = B + merge(L[D], L[E], DE)
= B + merge(DO, EO, DE)
= BD + merge(O, EO, E)
= BDEO

L[A] = A + merge(L[B], L[C], BC)
= A + merge(BDEO, CDFO, BC)
= AB + merge(DEO, CDFO, C)
= ABC + merge(DEO, DFO)
= ABCD + merge(EO, FO)
= ABCDEFO
```

所以创建 A 类的 `__init__` 和 `__new__` 方法调用顺序为：`A-->B-->C-->D-->E-->F-->O`。

读者可以通过上述代码，通过 `A.mro()` 或 `A.__mro__` 检验是否正确。

### 分析 C 为什么无法被创建

回到之前的问题，为什么 class C 是无法被创建的。

```python
O = object

class X(O): pass

class Y(O): pass

class A(X, Y):  pass

class B(Y, X):  pass

class C(A, B):  pass
```

继承树的结构如下：

```
            O
           / \
          /   \
        X      Y
       /  \  /  \
      /____\/____\
    A              B
      \           /
        \        /
          \    /
            C
```

很容易产生错误的认识，如果创建类 C 不会产生问题，类似 C++ 中的虚基类初始化顺序为：`O-->X-->Y-->X-->A-->B-->C`，但事实上确实无法创建 class C。

按照上述 C3 算法计算 L[C]：

```
L[O] = O
L[X] = XO
L[Y] = YO
L[A] = AXYO
L[B] = BYX0

L[C] = C + merge(L[A], L[B], AB)
= C + merge(AXYO, BYXO, AB)
= CA + merge(XYO, BYXO, B)
= CAB + merge(XYO, YXO)     # 无法继续计算
```

`merge(XYO, YXO)` 误解，因为 X/Y/O 三个元素都不满足以下两个条件：
1. 要不不存在 P 中
2. 要么是 P 中的第一个元素

所以 Python 无法确定其初始化的顺序，也就无法创建类 C。

## `__init__` 与 `__new__` 的区别

```python
class A:
    def __init__(self, *args, **kwargs):
        super(A, self).__init__(*args, **kwargs)

    def __new__(cls, *args, **kwargs):
        return super(A, cls).__new__(cls, *args, **kwargs)
```

| `__init__` | `__new__` |
| :-------: | :-----: |
| 初始化实例的属性 | 创建实例 |
| 没返回值 | 有返回值 |
| 不需要传递 self | 需要传递 cls |
| 后于 `__new__` 调用 | 先于 `__init__` 调用 |
| 实例方法，第一个参数是 self | 类方法，第一个参数是 cls |

大多数情况下，我们是不需要重写父类的 `__new__` 方法的，除非需要实现单例模式、不可变量等属性。元编程可以借助 `__new__` 实现，后面有机会写一篇关于 Python 元编程的文章。

`super().__init__()` 并没有携带 self 参数，说明 `super()` 调用返回的是一个实例。

而为什么 `super().__new__(cls)` 需要附带 cls 参数呢？

那首先得知道 `super()` 到底返回的是什么，在不同情况下调用有什么不同的表现？

我写了这个小 demo：

```python
from typing import Any


class A:
    def __new__(cls) -> Any:
        print('A.__new__')
        s = super()
        return s.__new__(cls)

    def __init__(self):
        print('A.__init__')
        s = super()
        s.__init__()


class B(A):

    def __new__(cls) -> Any:
        print('B.__new__')
        s = super()
        return s.__new__(cls)

    def __init__(self):
        print('B.__init__')
        s = super()
        s.__init__()


class C(B):

    def __new__(cls) -> Any:
        print('C.__new__')
        s = super()
        return s.__new__(cls)

    def __init__(self):
        print('C.__init__')
        s = super()
        s.__init__()


c = C()
```

通过打断点，逐个检查 super() 的返回值，以及每一个 `__new__` 方法中的 cls 参数 和 `__init__` 方法中的参数 self 的变化，我得出以下结论：

- `__new__` 方法先于 `__init__` 方法执行
- `super()` 似乎每次都返回相同的值
- `__new__` 返回不是本类的实例，`__init__` 方法也就无法被调用
- `__new__` 中方法的 cls 一直都是同一个 cls，也就是 cls 一直往下传递。我通过 `id(cls)` 来判断的，在 CPython 中，id 方法返回的是内存地址值，我发现 `id(cls)` 每次都返回同样的内容
- `__init__` 中方法的 self 一直都是同一个 self，验证方法与 `__new__` 一样

也就是说 `super()` 是找到 MRO 中下一个父类的 `__new__` 和 `__init__` 进行调用。

发现了没，Python 类中如果重写了某个父类方法 `fun(self)`，但是在某个时刻我们需要调用父类的方法 `fun`，该如何处理呢？

假设父类为 Base，子类为 Child。除了可以使用 `Base.fun(self)` 调用外。还可以 `super().fun()`，当然这种方案只能够调用在 MRO 中紧跟 Child 的类的方法，但是如果我们想多跳几级呢？设 Base 的唯一父类为 SuperBase，我们需要在 Child 中调用 SuperBase 的实例方法，我们可以 `super(Base, self).fun()`。当然在代码中应该避免这样的调用，因为下次阅读代码需要再次计算 MRO，为了代码的可读性，应该这样调用：`SuperBase.fun(self)`。

## new-style & old-style

在 stackoverflow 看到这样一个问题：
[old-style class 与 new-style class 区别](https://stackoverflow.com/questions/1848474/method-resolution-order-mro-in-new-style-classes/1848647#1848647)

```python
class A: x = 'a'

class B(A): pass

class C(A): x = 'c'

class D(B, C): pass

D.x     # 'a'
```

```python
class A(object): x = 'a'

class B(A): pass

class C(A): x = 'c'

class D(B, C): pass

D.x     # 'c'
```

上述的结果我在 Python3.6 中都无法复现，但是我在 Python2.7 中复现了。因为 "old-style class" 只存在于 Python2，Python3 中只有 new-style class。new-style class 是在 Python2.1 后引入的，以下声明方法决定其是 new or old style：

```python
# new
class A(object):    pass

# old
class A:    pass
```

## 不使用 super() 实现父类实例初始化

没有 super，怎么调用多级的父类 `__init__` 方法呢？

还记得我们有 `mro()` 方法，而且 super() 的初始化顺序就是按照 MRO 进行的。

如果没有 `super()`，可能需要写类似以下的代码：

```python
class Child(Base):
    def __init__(self):
        mro = type(self).mro()
        for next_class in mro[mro.index(Child)+1:]:
            if hasattr(next_class, '__init__'):
                # 调用实例方法
                next_class.__init__(self)
                break
```

在多继承中以下代码还能够正常运行吗？

可以。

最后举个**错误**例子：

```python
class A:
    def __init__(self):
        print('A.__init__')
        super(self.__class__, self).__init__()  # 1


class B(A):
    def __init__(self):
        print('B.__init__')
        super(self.__class__, self).__init__()  # 2
```

在 2 处调用 `super(self.__class__, self).__init__()` 时候传递的参数 self 是 B 的实例，所以传递到 `A.__init__` 1 处 self 依然是 B 的实例，`super(self.__class__, self).__init__()` 这条语句和 2 产生一样的效果，继续执行 `A.__init__` 最后导致栈溢出。

# 参考

[Things to Know About Python Super](https://www.artima.com/weblogs/viewpost.jsp?thread=236275)

[Python MRO](https://www.python.org/download/releases/2.3/mro/)
