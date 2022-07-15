---
layout: post
title: Python函数
date: 2022-07-15
tags: PYTHON
---

## Python函数基础

函数是由若干语句组成的语句块、函数名称、参数列表构成的，可完成一定功能的代码封装，它是组织代码的最小单元。

结构化编程使用函数对代码的做最基本的封装，实现代码的复用，减少冗余代码，使代码更加简洁，提高代码的可读性。

**函数分类**：
- 内建函数
- 库函数
- 自定义函数

**函数定义**

```
def 函数名(参数列表):
  函数块(代码块)
  [return 返回值]
```
- 函数名就是标识符
- 函数体实现具体代码逻辑
- 函数若没有return语句，会隐式返回None值
- 函数定义中的参数列表为形式参数，简称**形参**

**函数调用**

函数定义只是声明了一个函数，它不能被执行，需要被调用才能执行，调用的方式就是**函数名加上小括号**，如果有必要需要在括号内添加参数，参数调用时传入的值，称为**实参**。

```
def add(x,y) : # 函数定义，x,y为形参
    result = x+y   # 函数体，实现具体代码逻辑
    return result

out = add(4,5) # 函数调用， 4，5为实参
print(out)
```

#### 函数参数

函数的定义是要定义好形式参数，调用时提供足够的实际参数，一般来说形参喝实参个数要一直，可变参数除外。

1. 位置参数：`def f(x,y,z)`，按照参数定义顺序传入参数 `f(1,2,3)`  
2. 关键字参数：`def f(x,y)`，使用kv对方式传入参数`f(x=1,y=2)`或者f(y=1,x=2)
3. 可变位置参数：`def f(*args)`，在形参前添加一个`*`可用来接受多个实参，且将他们组织到tuple中
4. 可变关键字传参：`def(**kwargs)`，在形参前添加一个`**`可用来接受多个kv对，且将他们组织到字典中
5. 混合参数：`def f(x,y,*args,**kwargs)`
6. keyword-only参数：`def f(*args, x)`,python3 以后，在`*`号以后，出现一个的普通参数x只能通过kv对的方式传入,x成为keyword-only参数
7. Positional-only参数：`def fn(x, /)`,python3.8以后，支持仅位置参数，进位制参数不可以使用关键字参数传参


```

>>> def f(x,y,*args,**kwargs):
...     print(x,y,args,kwargs, sep="\n", end="\n")
...
>>> f(3,5,7,9,10,a=1,b=2)
3
5
(7, 9, 10)
{'a': 1, 'b': 2}    
```

> 混合参数使用时，位置参数必须要在关键之参数之前传入

#### 形参-定义缺省值

缺省值也叫默认值，使用在函数定义是，为形式参数提供一个缺省值，主要用于在未传够足够的实参时，对没有给顶的参数提供默认值，从而实现函数的简化调用。

```
def add(x=1, y=3):
   return x+y
```

通过缺省值，可以实现add函数使用 `add()`调用。

#### 参数解构

在给函数提供实参的时候，在可以使用可迭代对象前，使用`*`或者`**`来进行结构，提供其中所有的元素哦作为函数的实参
- `*`：结构成位置参数
- `**`：结构成关键字参数

```
>>> def add(*iterable):
...     result = 0
...     for x in iterable:
...         result += x
...     return result
...
>>> add(*(range(5)))
10
>>> add(*[1,3,5])
9

```

#### 函数返回值

返回值用来**结束函数调用，返回"返回值"**

- Python函数使用 return 语句返回返回值
- 所有的函数都有返回值，如果没有返回值，则隐式调用了 return None
- 一个函数可以又多个return语句，但是只有一条被执行
- 函数执行return语句，就会立即返回，其他语句不在执行


#### 函数作用域

一个标识符的课件范围，就是作用域，我么你一般说的作用域时指变量的作用域。**每个函数都会开辟一个新的作用域**

- 全局作用域：整个程序运行环境可见
- 局部作用域：在函数、类等内部可见

一般来说外部作用域变量在函数内部可见，但是函数内部作用域在函数外部不可见

```
>>> x = 5
>>> def foo():
...     print(x)
...
>>> foo()
5
```

#### 函数嵌套

在一个函数中定义了另一个函数，成为函数嵌套。内部函数inner不能直接在外部使用，会抛出`NameError`，因为它在函数外部不可见。

```
def outer():
    def inner():
        print("inner")
    print("outer")
    inner()
```

#### Global语句

在Python中，赋值即定义，只要有`x=`出现，这就是赋值语句。下面的函数foo内部定义了一个局部变量x，那么函数中所有x都是用这个局部变量x了，而 x=x+1,相当于使用局部变量x，但是这个x还没有复制完成，就被用来执行x+1操作，所以，此处报错 `UnboundLocalError`。
```
>>> x = 5
>>> def foo():
...     x+=1  # x=x+1 ,
...     print(x)
...
>>> foo()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 2, in foo
UnboundLocalError: local variable 'x' referenced before assignment
```

要解决这个问题，我们需要因为global语句，通过global关键字，我们可以把一个局部变量声明为全局变量。

```
>>> x = 5
>>> def foo():
...     global x  # 使用global 将本地变量x提升为全局变量，慎用
...     x+=1
...     print(x)
...
>>> foo()
6
```

#### 闭包

- **自由变量**：未在本地作用域中定义的变量，例如定义在内层函数外的外层函数作用域中的变量
- **闭包**：内层函数中引用到了外层函数中的自由变量，就形成了闭包

```
# Python 2 中实现闭包的方式
>>> def counter():  
...     c = [0]
...     def inc():
...         c[0] +=1  # 不会报错，修改的是引用，不会重新定义c，形成闭包
...         return c[0]
...     return inc # 返回一个inc函数对象，因为foo标记了所以不会释放
...
>>> foo = counter()  
>>> print(foo(), foo())
1 2

```

#### Nonlocal语句

将变量标记为不在本地作用域定义，而是在**上级的某一级局部作用域**中定义，但**不能是全局作用域**中定义。

```

>>> def counter():
...     count = 0
...     def inc():
...         nonlocal count #  声明count不是一个本地变量，从而实现闭包
...         count+=1
...         return count
...     return inc
...
>>> foo = counter()
>>> print(foo(), foo())
1 2
```


## 匿名函数

**匿名函数**: 没有名字的函数， Python中使用`Lambda`构建匿名函数

#### Lambda表达式

匿名函数使用lambda关键字定义，参数列表不需要小括号，多个参数使用`,`分割，冒号用来分割参数列表和表达式，不能出现return语句且不能出现等号，只能写在一行，所以也成为单行函数。

lambda函数一般用于为高阶函数传参

- 定义：`lambda [参数列表]: 表达式`
- 调用：`(lambda [参数列表]: 表达式)(参数)`

```
# 返回常量 返回10
(lambda :10)()

# 加法，带缺省值 返回8
(lambda x,y=3 :x+y)(5)

# keyword-only参数 返回35
(lambda x,*, y=30: x+y)(5)
(lambda x,*, y=30: x+y)(10, y=25)


# 可变参数，返回(4,5,6)
(lambda *args: args)(4,5,6) 
[(lambda *args: args)][0](4,5,6)

```

**使用lambda生成一个生成器**

```
a = (lambda *args: (x for x in args))(1,2,3)

for i in range(3):
    print(next(a))
```

```
>>> (lambda *args: [x for x in args])(1,2,3)
[1, 2, 3]
>>> (lambda *args: list(args))(1,2,3)
[1, 2, 3]
>>> (lambda *args: args)(1,2,3)
(1, 2, 3)
>>> (lambda *args: [args])(1,2,3)
[(1, 2, 3)]
```

**使用lambda得到一个集合**

```
import random

(lambda *args:{x %3 for x in args})( *[random.randint(1, 100) for x in range(10)])
```

#### Lambda在高阶函数sorted中的应用

```
>>> sorted(["a", "b", 12, "15", 19], key=str)
[12, '15', 19, 'a', 'b']
>>> sorted(["a", "b", 12, "15", 19], key=lambda x:str(x)) # 使用字符串排序
[12, '15', 19, 'a', 'b']


>>> sorted(["a", "b", 12, "15", 19], key=lambda x: int(x, 16) if isinstance(x, str) else x) # 使用数值排序
['a', 'b', 12, 19, '15']
```


## 递归函数

## 递归函数

#### 函数的执行流程
函数的活动跟栈有关，栈是**后进先出**的数据结构

#### 递归

函数直接或者间接的调用自身的方式就是递归
- 递归调用一定要有退出原则，否则会出现无限调用
- 递归的深度不宜过深
- Python对递归调用深度做了限制，可以通过`sys.getrecursionlimit()`查看
- 良好的递归调用需要注意边界问题，应该尽量避免递归

#### 使用递归打印斐波那契数列

```
>>> def fib(n):
...     if n<3:
...         return 1
...     else:
...         return fib(n-1)+fib(n-2)
...
>>> for i in range(10):
...     res = fib(i)
...     print(res, end=" ")
...
1 1 1 2 3 5 8 13 21 34 
```

## 高阶函数

函数时一等公民，函数可以是对象，也可以是可调用对象，其可以作为普通变量，函数的参数、函数返回值等。

**高阶函数**

高阶函数需要至少满足一下条件之一：
- 接收一个或者多个函数作为参数
- 输出一个函数

```
def counter(base):
    def inc(step=1):
        nonlocal base # base是闭包
        
        base += step # base=base+step 
        return base
    return inc
```

#### sorted 函数的工作原理

思路：
- 内奸函数sorted，接收一个可迭代对象，返回一个新的列表，可以使用reverse设置升序降序，可以通过key设置用于比较的函数
- 新建一个列表，遍历原列表，和新列表中的值一次比较，决定插入的数据在新列表的具体位置

```
# 实现排序
def sort(iterable, *, key=None, reverse=False):
    newlst = []
    for x in iterable:  
        for i, y in enumerate(newlst):
            if x < y: # 
                newlst.insert(i, x)
                break
        else:
            newlst.append(x) 
    return newlst
```

```
# 实现逆序
def sort(iterable, *, key=None, reverse=False):
    newlst = []
    for x in iterable:  
        for i, y in enumerate(newlst):
            if x < y: 
                newlst.insert(i, x)
                break
        else:  # 第一次为空列表，所以使用else子句
            newlst.append(x) 
    return newlst
```

```
# 实现可逆序
def sort(iterable, *, key=None, reverse=False):
    newlst = []
    for x in iterable:  
        for i, y in enumerate(newlst):
            # if x < y: 
            revs = (x>y) if reverse else (x<y) # 控制reverse
            if revs:
                newlst.insert(i, x)
                break
        else: # 
            newlst.append(x) 
    return newlst
```

```
# 实现 key 函数的功能
def sort(iterable, *, key=None, reverse=False): # key = str or key= int(x, base=16) if isinstance(x, str) else x
    newlst = []
    for x in iterable: 
        cx = key(x) if key else x
        for i, y in enumerate(newlst):
            cy = key(y) if key else y
            # if x < y: 
            revs = (x>y) if reverse else (x<y) # 控制reverse
            if revs:
                newlst.insert(i, x)
                break
        else: # 
            newlst.append(x) 
    return newlst
```

#### filter过滤函数

把符合条件的留下，不符合条件的筛除,返回惰性迭代器

- 如果filter函数的第一个参数为None，可迭代对象的所有元素按照本身的等效True或者False，如果等效为Flase则筛除
- 单参函数

```
def filter(function or None, iterable):
    pass 
```


```
>>> list(filter(None, range(-5, 5))) # 0等效False，所以筛除
[-5, -4, -3, -2, -1, 1, 2, 3, 4]
```

```
>>> list(filter(lambda x:x%3==0, [1,3,4,5,6,11,12])) #
[3, 6, 12]
```

#### map映射函数

对多个可迭代对象，按照指定的函数进行映射，返回一个迭代器，元素个数不变

```
def map(function, *Iterables) -> map object:
    pass
```

```
>>> list(map(lambda x: str(x+65), range(5)))
['65', '66', '67', '68', '69']

>>> list(map(lambda x: chr(x+65), range(5)))
['A', 'B', 'C', 'D', 'E']
```

```
>>> tuple(map(lambda x,y: (x,y), 'abcde', range(5)))
(('a', 0), ('b', 1), ('c', 2), ('d', 3), ('e', 4))

>>> dict(map(lambda x,y: (x,y), 'abcde', range(5)))
{'a': 0, 'b': 1, 'c': 2, 'd': 3, 'e': 4}
```


