---
{"dg-publish":true,"permalink":"/csdn/Pandas-数据处理/Pandas核心数据结构操作/","title":"Pandas核心数据结构操作","tags":["python","pandas"],"dg-note-properties":{"category":"Pandas / 数据处理","title":"Pandas核心数据结构操作","source":"csdn","created":"2021-08-11","tags":["python","pandas"],"url":"https://blog.csdn.net/weixin_45536921/article/details/119598544"}}
---



##### 核心数据结构操作

**列访问**


DataFrame的单列数据为一个Series。根据DataFrame的定义可以 知晓DataFrame是一个带有标签的二维数组，每个标签相当每一列的列名。


```python
import pandas as pd

d = {'one' : pd.Series([1, 2, 3], index=['a', 'b', 'c']),
     'two' : pd.Series([1, 2, 3, 4], index=['a', 'b', 'c', 'd'])}

df = pd.DataFrame(d)
print(df['one'])
print(df[['one', 'two']])

```
**列添加**


DataFrame添加一列的方法非常简单，只需要新建一个列索引。并对该索引下的数据进行赋值操作即可。


```python
import pandas as pd

data = {'Name':['Tom', 'Jack', 'Steve', 'Ricky'],'Age':[28,34,29,42]}
df = pd.DataFrame(data, index=['s1','s2','s3','s4'])
df['score']=pd.Series([90, 80, 70, 60], index=['s1','s2','s3','s4'])
print(df)

```
**列删除**


删除某列数据需要用到pandas提供的方法pop，pop方法的用法如下：


```python
import pandas as pd

d = {'one' : pd.Series([1, 2, 3], index=['a', 'b', 'c']), 
     'two' : pd.Series([1, 2, 3, 4], index=['a', 'b', 'c', 'd']), 
     'three' : pd.Series([10, 20, 30], index=['a', 'b', 'c'])}
df = pd.DataFrame(d)
print("dataframe is:")
print(df)

# 删除一列： one
del(df['one'])
print(df)

#调用pop方法删除一列
df.pop('two')
print(df)

```
**行访问**


如果只是需要访问DataFrame某几行数据的实现方式则采用数组的选取方式，使用 “:” 即可：


```python
import pandas as pd

d = {'one' : pd.Series([1, 2, 3], index=['a', 'b', 'c']), 
    'two' : pd.Series([1, 2, 3, 4], index=['a', 'b', 'c', 'd'])}

df = pd.DataFrame(d)
print(df[2:4])

```
**loc**方法是针对DataFrame索引名称的切片方法。loc方法使用方法如下：


```python
import pandas as pd

d = {'one' : pd.Series([1, 2, 3], index=['a', 'b', 'c']), 
     'two' : pd.Series([1, 2, 3, 4], index=['a', 'b', 'c', 'd'])}

df = pd.DataFrame(d)
print(df.loc['b'])
print(df.loc[['a', 'b']])

```
**iloc**和loc区别是iloc接收的必须是行索引和列索引的位置。iloc方法的使用方法如下：


```python
import pandas as pd

d = {'one' : pd.Series([1, 2, 3], index=['a', 'b', 'c']),
     'two' : pd.Series([1, 2, 3, 4], index=['a', 'b', 'c', 'd'])}

df = pd.DataFrame(d)
print(df.iloc[2])
print(df.iloc[[2, 3]])

```
**行添加**


```python
import pandas as pd

df = pd.DataFrame([['zs', 12], ['ls', 4]], columns = ['Name','Age'])
df2 = pd.DataFrame([['ww', 16], ['zl', 8]], columns = ['Name','Age'])

df = df.append(df2)
print(df)

```
**行删除**


使用索引标签从DataFrame中删除或删除行。 如果标签重复，则会删除多行。


```python
import pandas as pd

df = pd.DataFrame([['zs', 12], ['ls', 4]], columns = ['Name','Age'])
df2 = pd.DataFrame([['ww', 16], ['zl', 8]], columns = ['Name','Age'])
df = df.append(df2)
# 删除index为0的行
df = df.drop(0)
print(df)

```
**修改DataFrame中的数据**


更改DataFrame中的数据，原理是将这部分数据提取出来，重新赋值为新的数据。


```python
import pandas as pd

df = pd.DataFrame([['zs', 12], ['ls', 4]], columns = ['Name','Age'])
df2 = pd.DataFrame([['ww', 16], ['zl', 8]], columns = ['Name','Age'])
df = df.append(df2)
df['Name'][0] = 'Tom'
print(df)

```
**DataFrame常用属性**


编号属性或方法描述1```
axes
```返回 行/列 标签（index）列表。2```
dtype
```返回对象的数据类型(```
dtype
```)。3```
empty
```如果系列为空，则返回```
True
```。4```
ndim
```返回底层数据的维数，默认定义：```
1
```。5```
size
```返回基础数据中的元素数。6```
values
```将系列作为```
ndarray
```返回。7```
head()
```返回前```
n
```行。8```
tail()
```返回最后```
n
```行。实例代码：


```python
import pandas as pd

data = {'Name':['Tom', 'Jack', 'Steve', 'Ricky'],'Age':[28,34,29,42]}
df = pd.DataFrame(data, index=['s1','s2','s3','s4'])
df['score']=pd.Series([90, 80, 70, 60], index=['s1','s2','s3','s4'])
print(df)
print(df.axes)
print(df['Age'].dtype)
print(df.empty)
print(df.ndim)
print(df.size)
print(df.values)
print(df.head(3)) # df的前三行
print(df.tail(3)) # df的后三行

```