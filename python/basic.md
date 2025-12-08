### 本文档记录python基础笔记

#### 基础中的基础
- python很多最基础的数据类型，如（int）是自动确定类型，并且自动强制转换，自动变换类型的
- python中seq通常代表sequence(序列)，序列是指如列表、元组、字节串、字节数组之类的
- seq是可以迭代的，比如用for，可以直接使用相关数据对象自己的迭代器
- seq的运算
    - len(seq)
    - seq + seq
    - seq * 4
    - a in seq (会返回一个布尔值)
    - for a in seq: print(a)
    - seq[-2]是访问倒数第二个元素
    - seq负数索引-n是访问到倒数第n个元素，因为`负数，0, 正数`
    - seq索引切片是左闭右开的
    - python索引超出范围会自动截断
    - 当切片start比stop靠后，会直接返回空
    - 扩展切片指的是这样的切片形式：a[start:stop:step]，start默认为0，stop默认为尾部
```python console
 >>> a
 [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
 >>> a[0::-1]
 [0]
 >>> a[0:len(a):-1]
 []
```
```note
step为负数
当step为负数时，切片将其解释为从start出发以步长|step|逆序索引序列，此时，start和stop的截断依然遵循前述规则，但缺省发生一点变化，因为我们说过，在缺省的情况下，Python的行为是尽可能取最大区间，此时访问是逆序的，start应尽量取大，stop应尽量取小，才能保证区间最大，因此：

按照扩充索引范围的观点，start的缺省值是无穷大(​)，stop的缺省值是无穷小(​)
 >>> a
 [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
 >>> a[5::-1]
 [5, 4, 3, 2, 1, 0]
 >>> a[:4:-2]
 [9, 7, 5]
 >>> a[::-1]
 [9, 8, 7, 6, 5, 4, 3, 2, 1, 0]
```
- 列表（list）
    - 列表是可以进行区间插入的，比如list1[2:2] = list2表示在索引2前面加上list2
    - list[2:2]可以写+=，也可以写=
    - 从这个中括号的范围划定规则可以看出，[2:2]后面的2表示定位到索引2的前面。而从其他例子中分析得到，[x,y]切片是左闭右开的，就像vim的插入行为（i）一样
    - print会把列表当作一个列表对象输出，而不是输出元素
    - 列表有几个重要的操作函数或方法
        - cmp(list1,list2)
        - len(list)
        - max(list)
        - min(list)
        - list(seq)
        - list.append(obj)
        - list.count(obj)计算obj在list里的出现次数
        - list.extend(seq)在列表末尾一次性追加另一个序列中的多个值（用新列表扩展原来的列表）
        - list.index(obj)找出第一个匹配项的索引
        - list.insert(index, obj)将obj插入到index之前
        - list.pop(index)移除一个元素并返回其值，默认最后一个obj
        - list.remove(obj) 基本等同于 list.pop(list.index(obj))，但没有返回值
        - list.reverse()
        - list.sort(cmp=None,key=None,revrse=False)里面的参数是可调的
            - cmp会用指定的方法排序(python3中已废弃)
            - key用来比较的元素（python3主要用来制定比较逻辑）
            - reverse默认False(升序)

- python几乎所有数据类型都是对象，python可以说是像java一样的面向对象编程
    - 因此在代码中创建的数据可以用del删除
- 元组（tuple）
    - 元组与列表逻辑很相似，但是不可修改
    - 元组必须含有逗号
    - 元组不能直接赋值为空，但可以计算后为空
```python
#!/bin/bash
test = ('a',)
test2 = (1,2,3,4,)
test5 = test[0:0] + test2[1:1]
print(test5)
del test5
#print(test5)
#不能打印test5,因为test5被删除，访问会报错说没有test5
```
- 字典（这个不是seq）
    - dict = {'a':1,'b':2}
    - 值可以取任意数据类型，但键必须不可变
    - del dic['key1']删除键为key1的条目
    - dic.clear()删除字典所有条目
    - del dic删除字典
    - 字典表达式
        - cmp(dic1,dic2)
        - len(dic)计算键的总数
        - dic.copy()
        - dic.get(key,default=None)返回指定键的值，如果值不在字典中返回default值
        - dic.items()
        - pop(key[,default])
        - popitem()
```
>>> tinydict = {'a': 1, 'b': 2, 'b': '3'}
>>> tinydict['b']
'3'
>>> tinydict
{'a': 1, 'b': '3'}
```
- time模块
    - import time
    - time.time()返回时间戳（时间戳无法表示1970年以前，无法表示遥远的将来（max=2038））
    - time.localtime(time.time())会显示本地时间
    - localtime = time.asctime(time.localtime(time.time()))格式化的本地时间
#### python函数
- python函数是自动变换返回类型的，主要看你return写什么
- python的数据作用域与生存时长与c/cpp相似
- 可以用参数名来匹配参数，防止乱序
- 不要把参数名匹配参数与位置匹配参数混用
- 不定长参数
    - 加了星号（*）的变量名会存放所有未命名的变量参数。
```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
# 可写函数说明
def printinfo( arg1, *vartuple ):
   "打印任何传入的参数"
   print("输出: ")
   print(arg1)
   for var in vartuple:
      print(var)
   return
 
# 调用printinfo 函数
printinfo( 10 )
printinfo( 70, 60, 50 )
```
- 匿名函数lambda
