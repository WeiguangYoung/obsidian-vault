---
{"dg-publish":true,"permalink":"/csdn/Pandas-数据处理/Pandas基础/","title":"Pandas基础","tags":["python","pandas"],"dg-note-properties":{"category":"Pandas / 数据处理","title":"Pandas基础","source":"csdn","created":"2021-08-11","tags":["python","pandas"],"url":"https://blog.csdn.net/weixin_45536921/article/details/119598412"}}
---



#### pandas介绍

Python Data Analysis Library


pandas是基于NumPy 的一种工具，该工具是为了解决数据分析任务而创建的。Pandas 纳入 了大量库和一些标准的数据模型，提供了高效地操作大型结构化数据集所需的工具。


#### pandas核心数据结构

数据结构是计算机存储、组织数据的方式。 通常情况下，精心选择的数据结构可以带来更高的运行或者存储效率。数据结构往往同高效的检索算法和索引技术有关。


##### Series

Series可以理解为一个一维的数组，只是index名称可以自己改动。类似于定长的有序字典，有Index和 value。


```python
import pandas as pd
import numpy as np

# 创建一个空的系列
s = pd.Series()
# 从ndarray创建一个系列
data = np.array(['a','b','c','d'])
s = pd.Series(data)
s = pd.Series(data,index=[100,101,102,103])
# 从字典创建一个系列	
data = {'a' : 0., 'b' : 1., 'c' : 2.}
s = pd.Series(data)
# 从标量创建一个系列
s = pd.Series(5, index=[0, 1, 2, 3])

```
访问Series中的数据：


```python
# 使用索引检索元素
s = pd.Series([1,2,3,4,5],index = ['a','b','c','d','e'])
print(s[0], s[:3], s[-3:])
# 使用标签检索数据
print(s['a'], s[['a','c','d']])

```

##### DataFrame

DataFrame是一个类似于表格的数据类型，可以理解为一个二维数组，索引有两个维度，可更改。DataFrame具有以下特点：


- 潜在的列是不同的类型
- 大小可变
- 标记轴(行和列)
- 可以对行和列执行算术运算

```python
import pandas as pd

# 创建一个空的DataFrame
df = pd.DataFrame()
print(df)

# 从列表创建DataFrame
data = [1,2,3,4,5]
df = pd.DataFrame(data)
print(df)
data = [['Alex',10],['Bob',12],['Clarke',13]]
df = pd.DataFrame(data,columns=['Name','Age'])
print(df)
data = [['Alex',10],['Bob',12],['Clarke',13]]
df = pd.DataFrame(data,columns=['Name','Age'],dtype=float)
print(df)
data = [{'a': 1, 'b': 2},{'a': 5, 'b': 10, 'c': 20}]
df = pd.DataFrame(data)
print(df)

# 从字典来创建DataFrame
data = {'Name':['Tom', 'Jack', 'Steve', 'Ricky'],'Age':[28,34,29,42]}
df = pd.DataFrame(data, index=['s1','s2','s3','s4'])
print(df)
data = {'one' : pd.Series([1, 2, 3], index=['a', 'b', 'c']),
        'two' : pd.Series([1, 2, 3, 4], index=['a', 'b', 'c', 'd'])}
df = pd.DataFrame(data)
print(df)

```