### 本文档记录python基础笔记

#### 基础中的基础
- python很多最基础的数据类型，如（int）是自动确定类型，并且自动强制转换，自动变换类型的
- python中seq通常代表sequence(序列)，序列是指如列表、元组、字节串、字节数组之类的
- seq是可以迭代的，比如用for，可以直接使用相关数据对象自己的迭代器
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
- 
