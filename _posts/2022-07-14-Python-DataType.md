---
layout: post
title: Python常用的内置数据结构汇总
date: 2022-07-14
tags: PYTHON
---

## Python 内置数据结构

Python 常用的内建数据结构有一下几种
- 线性数据结构
  - 列表list、元组tuple
  - 字符串str、字节序列bytes、bytearray
- 非线性数据结构
  - 集合set
  - 字典dict

## 列表-List

列表是**线性**的，有序的,**可变**，不可Hash的，**引用类型**的数据结构

#### 列表初始化

- 初始化空列表：`lst = list()` or `lst = []`
- 将可迭代对象转换为列表结构：`lst = list(iterable)`

#### 索引

列表通过索引访问 `lst[index]`, index 可正可负，索引不可以超界，否则引发异常 `IndexError`
- 正数范围：`[0,len(lst)-1]`
- 负数范围：`[-len(lst), -1]`
- `len()`：计算索引的长度并且返回


通过索引定位元素的时间复杂度为 O(1)


#### 查询元素

**根据元素查询**

- `index(value, [start, [stop]])`：通过 value 查询指定区间内的元素，查询到则返回索引，否则报错 `ValueError`, 时间复杂度为 O(n)
- `count(value)`：返回列表中匹配到的value的次数,时间复杂度为 O(n)

#### 修改元素

通过索引定位元素，然后直接复制，就可以修改原来的元素，需要注意的是索引不能超界。

```
>>> lst = list(range(5)) 
>>> lst
[0, 1, 2, 3, 4]
>>> lst[2] = 'a' 
>>> lst
[0, 1, 'a', 3, 4]
```

#### 增加元素

**增加单个元素**

- `append(object)`：尾部追加，就地修改，返回 None,时间复杂度为 `O(1)`
- `insert(index, object)`：在指定为之插入元素，返回 None，时间复杂度为 `O(n)`

**增加多个元素**

- `extend(iterable)`：将可迭代对象元素追加进来，就地修改
- `+`：等价于 `__add__()`方法，将两个列表元素连接起来，返回新的列表，原列表不变
- `*`：重复操作，将本列表元素，重读 n 次，返回新的列表

#### 删除元素

**根据元素删除**

- `remove(value)`：从左至右查找匹配到的第一个元素，返回None,否则报错 `ValueError`

**根据索引删除**
- `pop([index])`：从指定索引处弹出一个元素，不指定索引则从尾部弹出一个元素，返回被弹出元素，索引超界抛出 `IndexError`
- `clear()`：清空所有元素，返回列表的长度,剩下一个空列表，返回None

#### 反转元素

- `reverse()`：将列表就地反转，返回None，效率极低，建议倒着读

#### 排序

- `sort(key=None, reverse=False)`：对元素进行排序，默认为升序

#### 成员操作

- `in`：判断一个元素在不在可迭代对象中，返回值为布尔型

```
>>> lst
[0, 1, 'a', 3, 4]
>>> 1 in lst
True
```

#### 列表的复制


一般情况下，大多数语言提供的默认复制行为都是浅拷贝


- `shadow copy`：影子拷贝、浅拷贝遇到引用类型的数据，仅仅复制引用指针


```
>>> a = [1, 2, ["a", "b"], 3]
>>> b = a.copy() # 浅拷贝
>>> a == b
True
>>> b[2][0] = "A" # b[2]元素是一个列表为引用类型，所以a列表元素也发送变化
>>> a == b
True
>>> a is b
False
>>> a
[1, 2, ['A', 'b'], 3]
>>> b
[1, 2, ['A', 'b'], 3]

>>> b[2] = "C"  # "C" 为简单类型，所以 a列表元素没有发生变化
>>> a == b
False
>>> b
[1, 2, 'C', 3]
>>> a
[1, 2, ['A', 'b'], 3]

```

- `deep copy`：深拷贝，会递归繁殖一定的深度

```

>>> import copy
>>> a = [1, 2, ["a", "b"], 3]
>>> b = copy.deepcopy(a)
>>> a == b
True
>>> a[2][0] = "A"   # 深拷贝，会拷贝出一个新的引用，是一个独立的个体
>>> a == b
False
>>> a
[1, 2, ['A', 'b'], 3]
>>> b
[1, 2, ['a', 'b'], 3]

```

## 元组-Tuple

元组是**有序**的**不可变**的数据结构，可Hash

#### 初始化

- 空元组：`t1 = ()` or `t1 = tuple()`
- 将可迭代对象组织成元组：`t1 = tuple(range(5))` or `t1 = tuple([1,2,3,4])`or `t1 = (1,)`

#### 索引

元组的索引结构规则跟列表一样，同样不可以越界

通过索引定位元素的时间复杂度为 O(1)

#### 查询

- `index(value, [start, [stop]])`：通过 value 查询指定区间内的元素，查询到则返回索引，否则报错 `ValueError`, 时间复杂度为 O(n)
- `count(value)`：返回元组中匹配到的value的次数,时间复杂度为 O(n)

#### 增删改

元组是不可变数据结构，所以不存在这些操作。

如果元组中放的是引用类型指针，则元组内的引用指针所指向的数据可以被修改

```
>>> t = (1, 2, [3, 4], 5)
>>> t[2][1] = 'B'
>>> t
(1, 2, [3, 'B'], 5)
```

## 字符串-Str

字符串是一个**有序**的, **不可变的**字符的合集，通常使用单双三引号引住的字符序列

#### 初始化

```

>>> s = 'string'
>>> s1 = 'string'
>>> s2 = "string"
>>> s3 = """this'is a string"""
>>> s4 = r"hello \t world"  # r或者R前缀，保留字符原来的意思，没有转义
>>> name="tom"; age =18  # python 代码将多行代码写在一行用 ; 分割，不建议这么使用
>>> s5 = f'{name}, {age}' # python3.6 开始可以使用变量插入值
>>> s5
'tom, 18'

```

#### 字符串索引

字符串是序列，支持索引访问，但是不可以修改元素

#### 字符串连接

- `str1 + str2`：将两个字符串连接起来，返回一个新的字符串
- `seq.join(iterable)`：使用指定的分隔符,将iterable中的元素拼接起来,iterable中得元素必须为字符串， 否则报错 `TypeError`

```
>>> lst = ['a', 'b', 'c']
>>> '_'.join(lst)
'a_b_c'
>>> lst = [1, 2, 3, 4]
>>> "_".join(map(str, lst))
'1_2_3_4'

>>> lst2 = [1, 2, 3, 4]
>>> "_".join(lst2)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: sequence item 0: expected str instance, int found

```

#### 字符串查找

- `find(sub [,start[,end]])`：从左至右，在指定得区间 `[start,end)`,查找子串，找到返回正索引，否则返回 `-1`
- `rfind(sub [,start[,end]])`：从右至左，在指定得区间 `[start,end)`,查找子串，找到返回正索引，否则返回 `-1`
- `index(sub [,start[,end]])`：从左至右，在指定得区间 `[start,end)`,查找子串，找到返回正索引，否则返回 `ValueError`
- `rfind(sub [,start[,end]])`：从右至左，在指定得区间 `[start,end)`,查找子串，找到返回正索引，否则返回 `ValueError`
- `count(sub [,start[,end]])`：从左至右，在指定得区间 `[start,end)`，统计字串出现的次数
- `len(string)`：返回字符串的长度

#### 字符串分割

- `split(seq=None, maxsplit=-1) -> list of strings`
  - 从左至右，使用指定的分隔符seq分割字符串，默认分隔符为空格，maxsplit 为分割的次数,立即放回列表,不保留分隔符

- `rsplit(seq=None, maxsplit=-1) -> list of strings`
  - 从右至左，使用指定的分隔符seq分割字符串，默认分隔符为空格，maxsplit 为分割的次数,立即放回列表,不保留分隔符

- `splitlines([keepends]) -> list of strings`
  - 按照行来切割字符串，行分隔符包括 `\n,\r\n, \r`等，keepends 指保留行分隔符，默认为False不保留
  
- `partition(seq) -> (head,seq, tail)`
  - 从左至右，将字符串分为一个三元组，seq字符串必须指定，如果没有找到字符串，则seq跟tail为空元素
  
- `partition(seq) -> (head,seq, tail)`
  - 从右至左，将字符串分为一个三元组，seq字符串必须指定，如果没有找到字符串，则head跟seq为空元素
  
  
#### 字符串分割

- `replace(old, new[, count]) -> str`：使用新串替换老串，count为替换次数，不指定则全部替换，返回新串

#### 字符串移除

- `strip([chars]) -> str`：在字符串两端去除指定的字符集chars中的所有字符，如果没有指定chars则去除两边的空白符

#### 首尾判断

- `endswith(suffix[, start[,end]]) -> bool`：在指定的区间判断字符串是不是以suffix结尾
- `endswith(prefix[, start[,end]]) -> bool`：在指定的区间判断字符串是不是以prefix开头


#### 其他函数

- upper():大写
- lower()：小写
- swapcase()：交换大小写
- isalnum()：是否是字母或数字
- isdigit()：是否全部是数字(0-9)
- isdentifier()：是否是字幕和下划线开头

### 字符对齐

```
In [12]: s="abcd"

In [13]: "{:<8}".format(s)  # 左对齐，右边补空格
Out[13]: 'abcd    '

In [14]: "{:>8}".format(s)  # 右对齐，左边补空格
Out[14]: '    abcd'

In [15]: "{:^8}".format(s)  # 中间对齐，两边补空格
Out[15]: '  abcd  '

In [16]: "{:#<8}".format(s)
Out[16]: 'abcd####'

In [17]: "{:#>8}".format(s)
Out[17]: '####abcd'

In [18]: "{:#^8}".format(s)
Out[18]: '##abcd##'
```

## 字节序列-Bytes

字节是**不可变**的序列

#### 编码解码

- **编码** `str => bytes`：使用`encode()`将字符串转变为字节序列的过程
- **解码** `bytes => str`：使用`decode()`将字符串转变为字节序列的过程

一个字节等于8bit

一个UTF8编码的中文字符占用3个字节

一个GBK编码的中文字符占用2个字节

```
>>> 'abc'.encode()
b'abc'
>>> '中国'.encode()
b'\xe4\xb8\xad\xe5\x9b\xbd'
>>> '中国'.encode().decode()
'中国'
>>> len('中国'.encode("UTF8"))
6
>>> len('中国'.encode("GBK"))
4
```

#### 常见的ASCII编码

- `\x00`：`[NULL]` 表中的第一项，C语言中字符串结束符
- `\x09`： `\t`， 指标符号
- `\x0d\0a`：`\r\n`
- `\x30 ~\x39`：字符串`0~9`
- `\x41`：字符串`A`，对应的十进制为65
- `\x61`：字符串`a`, 对应十进制为97


```
>>> 'a\x0d \x09 \x0a \x31 \x61'
'a\r \t \n 1 a'
```

```
>>> 'a' > 'A'
True
>>> 'AA' > 'Aa'
False
```

#### 字节序列初始化

- 空序列：`bytes()`
- 指定字节的bytes：`bytes(int)`
- 使用可迭代的int对象创建：`bytes(iterable_of_ints)`
- 使用字符串创建：`bytes(string, encoding[,errors])`， 等价于string.encode()


#### 字节索引

```
>>> b'Aa'[1]  # 返回 int，指定本字节对应的十进制数
97
```

#### 字节的类构建方法

```
>>> bytes.fromhex('61 62 63 09 6a 6b') 
b'abc\tjk'

>>> 'abc\tjk'.encode().hex()
'616263096a6b'
```

## 字节数组-bytearray

字节数组是可变的

#### 字节数组初始化

- 空序列：`bytearray()`
- 指定字节的bytes：`bytearray(int)`
- 使用可迭代的int对象创建：`bytearray(iterable_of_ints)`
- 使用字符串创建：`bytearray(string, encoding[,errors])`， 等价于string.encode()

#### 字节数组的操作方法

字节数组类似与列表，可以使用列表中的所有方法, 下面方法int方位为 `[0, 255]`

- append(int)
- insert(index, int)
- extend(iterable_of_ints)
- pop(index=-1)
- remove(value)
- clear()
- reverse()

## 集合-Set

集合是**可变的**、 **无序的**、**不重复**、**不可索引**的元素的集合

#### 集合初始化

- 空集合：`set()` or `s = {}`
- `set(iterable)` 

#### 增加元素

**单元素增加**
- add(elem)：增加一个元素到集合中，如果元素已经存在则什么都不做

**批量增加**
- update(iterable)：合并其他元素到集合中，就地修改

#### 删除元素

- remove(elem)：移除一个元素，返回值为None，元素不存在报错 `KeyError`
- discard(elem)：移除一个元素，返回值为None，元素不存在不报错 
- pop()：移除并返回任意一个元素，集合为空则抛出`KeyError`
- clear()：移除所有元素

#### 集合相关概念

略

## 字典-Dict

字典是**可变的**、**无序的**、**key不重复**的键值对集合

#### 字典初始化

- 使用一个或者多个k=v键值对初始化：`dict(**kwargs)`
- 使用一个或者多个二元结构的可迭代对象构成：`dict([(x1, y1),..., (xn, yn)], **kwargs)`
- 使用一个字典构成另一个字典：`dict(Dict, **kwargs)`

```
d = dict( ( [(1,2), (3,4), [5, 6]]),  name="jerry", age=18) # k=v必须放在最后
>>> d
{1: 2, 3: 4, 5: 6, 'name': 'jerry', 'age': 18}

>>> d1 = {"A": 1, "B":2}
>>> d2 = dict(d1, name="tom", age=28)  # k=v必须放在最后
>>> d2
{'A': 1, 'B': 2, 'name': 'tom', 'age': 28}

```

#### 字典的访问

- `d[key]`：返回对应的value，key不错在报错`KeyError`
- `d.get(key[,default])`：返回对应的value， key不存在返回default，没有缺省值则返回None
- `d.setdefault(key[,default])`：返回对应的value， key不存在则添加kv对，value设置为default，如果没有缺省值则value为None

#### 字典新增

- `d[key] = value`：key存在在修改值为value，不存在则添加新的kv对
- `d.update(iterable, **kwargs)`： iterable 为一个可迭代的二元组或者一个字典

```
>>> d = {"a": 1}
>>> d.update((("A",2), ["B", 3]), name="jerry")
>>> d
{'a': 1, 'A': 2, 'B': 3, 'name': 'jerry'}
>>> d2 = {"C": 4, "D":5}
>>> d.update(d2, age=18)
>>> d
{'a': 1, 'A': 2, 'B': 3, 'name': 'jerry', 'C': 4, 'D': 5, 'age': 18}

```

#### 字典的删除

- `d.pop(key[,default])`：key存在，移除并返回对应的value，key不存在则返回default，如果没有设置default，则报错`KeyError`
- `d.popitem()`：移除并且返回一个任意的键值对组成的元组，空字典则抛出`KeyError`

#### 字典的遍历

- **`d.keys()`：遍历Key**

```
d = {"a": 1, "b": 2, "c": 3}

# 方法1
for k in d: 
    print(k)

# 方法2
for k in d.keys():
    print(k)
   
# 方法3
for k, _ in d.items():
    print(k)
```

- **`d.values()`：遍历value**

```
d = {"a": 1, "b": 2, "c": 3}

# 方法1
for k in d: 
    print(d[k])

# 方法2
for v in d.values():
    print(v)

# 方法3
for _, v in d.items():
    print(v)
```

- **`d.items()`遍历Item**

```
d = {"a": 1, "b": 2, "c": 3}

for item in d.items():
    print(item)  # ("a", 1) 返回二元组
    print(item[0], item[1])
    
for k, v in d.items():
    print(k, v)
```

#### 字典的遍历删除

在使用字典的 `keys`, `values`, `items` 方法遍历字典的时候不可以改变字典的长度(size),所以我们不能直接使用方法1删除字典，应该使用方法2，或者方法3，方法2的方式不如直接clear.

```
# 方法1-错误的做法
d = {"a": 1, "b": 2, "c": 3}

for k in d.keys(): # RuntimeError: dictionary changed size during iteration
    d.pop(k)

```

```
# 方法2 等价于 d.clear(), 建议使用d.clear()

d = {"a": 1, "b": 2, "c": 3}
while len(d):
    d.popitem()
```

```
# 方法3 - 使用一个列表来保存 key，而后遍历key删除

d = {"a": 1, "b": 2, "c": 3}

keys = []

for k in d.keys():
    keys.append(k)
    
for i in keys:
    d.pop(i)
```

#### 字典的有序性

Python 3.6+ 记录了字典的录入顺序，遍历的时候，就是按照这个顺序进行遍历的，如果非要使用有序比遍历，建议使用 OrderedDict。

```
from collections import OrderedDict

od = OrderedDict()
for i in "abcd":
    od[i] = int(i, base=16)
print(od) 

# OrderedDict([('a', 10), ('b', 11), ('c', 12), ('d', 13)])
```

#### defaultdict 类

python 提供defaultdict类，defaultdict(default_factory) 构建一个特殊字典，初始化时传入一个工程函数，当访问一个不存在的key时，会调用这个工厂函数返回值和key凑成一个键值对。

```
from collections import defaultdict

d2 = defaultdict(list)
for k in 'abcd':
    for v in range(5):
        d2[k].append(v) # k:list()
        
# d2 = defaultdict(list,
            {'a': [0, 1, 2, 3, 4],
             'b': [0, 1, 2, 3, 4],
             'c': [0, 1, 2, 3, 4],
             'd': [0, 1, 2, 3, 4]})
```


