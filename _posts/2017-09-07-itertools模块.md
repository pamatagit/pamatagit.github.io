---
layout: post
title: itertools模块
date: 2017-09-07
category: "python"
---

# Python标准库：itertools

## 简介

官方描述：Functional tools for creating and using iterators。即用于创建高效迭代器的工具。

## itertools.chain(*iterable)

将多个序列作为一个单独的序列返回，该序列只能用来迭代。例如：

```
import itertools
for each in itertools.chain('i', 'love', 'python'):
    print each
```

输出：

> i
> l
> o
> v
> e
> p
> y
> t
> h
> o
> n

## itertools.combinations(iterable, r)

返回指定长度的“组合”。例如：

```
import itertools
for each in itertools.combinations('abc', 2):
    print each
```

输出：

> ('a', 'b')
> ('a', 'c')
> ('b', 'c')

## itertools.combinations_with_replacement(iterable, r)

返回指定长度的“组合”，组合内元素可重复。例如：

```
import itertools
for each in itertools.combinations_with_replacement('abc', 2)
```

输出：

> ('a', 'a')
> ('a', 'b')
> ('a', 'c')
> ('b', 'b')
> ('b', 'c')
> ('c', 'c')

## itertools.product(*iterable[,repeat])

从每个iterable中取一个元素组合，重复repeat次，可理解为笛卡尔乘积。例如：

```
import itertools
for each in itertools.product('ab', 'cd', repeat=2):
    print each
```

输出：

> ('a', 'c', 'a', 'c')
> ('a', 'c', 'a', 'd')
> ('a', 'c', 'b', 'c')
> ('a', 'c', 'b', 'd')
> ('a', 'd', 'a', 'c')
> ('a', 'd', 'a', 'd')
> ('a', 'd', 'b', 'c')
> ('a', 'd', 'b', 'd')
> ('b', 'c', 'a', 'c')
> ('b', 'c', 'a', 'd')
> ('b', 'c', 'b', 'c')
> ('b', 'c', 'b', 'd')
> ('b', 'd', 'a', 'c')
> ('b', 'd', 'a', 'd')
> ('b', 'd', 'b', 'c')
> ('b', 'd', 'b', 'd')

## itertools.permutations(iterable[,r])

返回长度为r的排列，若r不指定，则默认为iterable参数的长度。例如：

```
import itertools
for each in itertools.permutations('abc', 2):
    print each
```

输出：

> ('a', 'b')
> ('a', 'c')
> ('b', 'a')
> ('b', 'c')
> ('c', 'a')
> ('c', 'b')

## itertools.compress(data,selector)

返回selector为True的data对应的元素。例如：

```
import itertools
for each in itertools.compress('abcd',[1,0,1,0]):
    print each
```

输出：

> a
> c

## itertools.count(start=0, step=1)

返回以start开始，step递增的序列，无限递增。例如：

```
import itertools
for each in itertools.count(start=0,step=2):
    print each
```

输出：

> 1
> 3
> 5
> 7
> 9
> .
> .

## itertools.cycle(iterable)

将迭代器进行无限迭代。例如：

```
import itertools
for each in itertools.cycle('ab'):
    print each
```

输出：

> a
> b
> a
> b
> .
> .

## itertools.dropwhile(predicate, iterable)

直到predicate为真，就返回iterable后续数据， 否则drop掉 例如：

```
import itertools
for each in itertools.dropwhile(lambda x: x<5, [2,1,6,8,2,1]):
    print each
```

输出：

> 6
>
> 8
>
> 2
>
> 1

## itertools.groupby(iterable[,key])

返回一组（key,itera）,key为iterable的值，itera为等于key的所有项 例如:

```
import itertools
for key, vale in itertools.groupby('aabbbc'):
   print key, list(vale)
```

输出:

> a ['a', 'a']
>
> b ['b', 'b', 'b']
>
> c ['c']

## itertools.repeat(object,[,times])

不停的返回object对象，如果指定了times,则返回times次 例如：

```
import itertools
for value in itertools.repeat('a', 2):
   print value
```

输出:

> a
>
> a