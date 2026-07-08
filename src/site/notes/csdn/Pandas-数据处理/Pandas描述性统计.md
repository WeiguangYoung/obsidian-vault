---
{"dg-publish":true,"permalink":"/csdn/Pandas-数据处理/Pandas描述性统计/","title":"Pandas描述性统计","tags":["python","pandas","numpy"],"dg-note-properties":{"category":"Pandas / 数据处理","title":"Pandas描述性统计","source":"csdn","created":"2021-08-11","tags":["python","pandas","numpy"],"url":"https://blog.csdn.net/weixin_45536921/article/details/119599305"}}
---



#### pandas描述性统计

数值型数据的描述性统计主要包括了计算数值型数据的完整情况、最小值、均值、中位 数、最大值、四分位数、极差、标准差、方差、协方差等。在NumPy库中一些常用的统计学函数也可用于对数据框进行描述性统计。


```python
np.min	最小值 
np.max	最大值 
np.mean	均值 
np.ptp	极差 
np.median	中位数 
np.std	标准差 
np.var	方差 
np.cov	协方差

```
实例：


```python
import pandas as pd
import numpy as np

# 创建DF
d = {'Name':pd.Series(['Tom','James','Ricky','Vin','Steve','Minsu','Jack',
   'Lee','David','Gasper','Betina','Andres']),
   'Age':pd.Series([25,26,25,23,30,29,23,34,40,30,51,46]),
   'Rating':pd.Series([4.23,3.24,3.98,2.56,3.20,4.6,3.8,3.78,2.98,4.80,4.10,3.65])}

df = pd.DataFrame(d)
print(df)
# 测试描述性统计函数
print(df.sum())
print(df.sum(1))
print(df.mean())
print(df.mean(1))

```
pandas提供了统计相关函数：


1```
count()
```非空观测数量2```
sum()
```所有值之和3```
mean()
```所有值的平均值4```
median()
```所有值的中位数5```
std()
```值的标准偏差6```
min()
```所有值中的最小值7```
max()
```所有值中的最大值8```
abs()
```绝对值9```
prod()
```数组元素的乘积10```
cumsum()
```累计总和11```
cumprod()
```累计乘积pandas还提供了一个方法叫作describe，能够一次性得出数据框所有数值型特征的非空值数目、均值、标准差等。


```python
import pandas as pd
import numpy as np

#Create a Dictionary of series
d = {'Name':pd.Series(['Tom','James','Ricky','Vin','Steve','Minsu','Jack',
   'Lee','David','Gasper','Betina','Andres']),
   'Age':pd.Series([25,26,25,23,30,29,23,34,40,30,51,46]),
   'Rating':pd.Series([4.23,3.24,3.98,2.56,3.20,4.6,3.8,3.78,2.98,4.80,4.10,3.65])}

#Create a DataFrame
df = pd.DataFrame(d)
print(df.describe())
print(df.describe(include=['object']))
print(df.describe(include=['number']))

```

#### pandas排序

*Pandas*有两种排序方式，它们分别是按标签与按实际值排序。


```python
import pandas as pd
import numpy as np

unsorted_df=pd.DataFrame(np.random.randn(10,2),
                         index=[1,4,6,2,3,5,9,8,0,7],columns=['col2','col1'])
print(unsorted_df)

```
**按行标签排序**


使用```
sort_index()
```方法，通过传递```
axis
```参数和排序顺序，可以对```
DataFrame
```进行排序。 默认情况下，按照升序对行标签进行排序。


```python
import pandas as pd
import numpy as np

d = {'Name':pd.Series(['Tom','James','Ricky','Vin','Steve','Minsu','Jack',
   'Lee','David','Gasper','Betina','Andres']),
   'Age':pd.Series([25,26,25,23,30,29,23,34,40,30,51,46]),
   'Rating':pd.Series([4.23,3.24,3.98,2.56,3.20,4.6,3.8,3.78,2.98,4.80,4.10,3.65])}
unsorted_df = pd.DataFrame(d)
# 按照行标进行排序
sorted_df=unsorted_df.sort_index()
print (sorted_df)
# 控制排序顺序
sorted_df = unsorted_df.sort_index(ascending=False)
print (sorted_df)

```
**按列标签排序**


```python
import numpy as np

d = {'Name':pd.Series(['Tom','James','Ricky','Vin','Steve','Minsu','Jack',
   'Lee','David','Gasper','Betina','Andres']),
   'Age':pd.Series([25,26,25,23,30,29,23,34,40,30,51,46]),
   'Rating':pd.Series([4.23,3.24,3.98,2.56,3.20,4.6,3.8,3.78,2.98,4.80,4.10,3.65])}
unsorted_df = pd.DataFrame(d)
# 按照列标签进行排序
sorted_df=unsorted_df.sort_index(axis=1)
print (sorted_df)

```
**按某列值排序**


像索引排序一样，```
sort_values()
```是按值排序的方法。它接受一个```
by
```参数，它将使用要与其排序值的```
DataFrame
```的列名称。


```python
import pandas as pd
import numpy as np

d = {'Name':pd.Series(['Tom','James','Ricky','Vin','Steve','Minsu','Jack',
   'Lee','David','Gasper','Betina','Andres']),
   'Age':pd.Series([25,26,25,23,30,29,23,34,40,30,51,46]),
   'Rating':pd.Series([4.23,3.24,3.98,2.56,3.20,4.6,3.8,3.78,2.98,4.80,4.10,3.65])}
unsorted_df = pd.DataFrame(d)
# 按照年龄进行排序
sorted_df = unsorted_df.sort_values(by='Age')
print (sorted_df)
# 先按Age进行升序排序，然后按Rating降序排序
sorted_df = unsorted_df.sort_values(by=['Age', 'Rating'], ascending=[True, False])
print (sorted_df)

```

#### pandas分组

在许多情况下，我们将数据分成多个集合，并在每个子集上应用一些函数。在应用函数中，可以执行以下操作 :


- *聚合* - 计算汇总统计
- *转换* - 执行一些特定于组的操作
- *过滤* - 在某些情况下丢弃数据

```python
import pandas as pd

ipl_data = {'Team': ['Riders', 'Riders', 'Devils', 'Devils', 'Kings',
         'kings', 'Kings', 'Kings', 'Riders', 'Royals', 'Royals', 'Riders'],
         'Rank': [1, 2, 2, 3, 3,4 ,1 ,1,2 , 4,1,2],
         'Year': [2014,2015,2014,2015,2014,2015,2016,2017,2016,2014,2015,2017],
         'Points':[876,789,863,673,741,812,756,788,694,701,804,690]}
df = pd.DataFrame(ipl_data)
print(df)

```

##### 将数据拆分成组

```python
# 按照年份Year字段分组
print (df.groupby('Year'))
# 查看分组结果
print (df.groupby('Year').groups)

```

##### 迭代遍历分组

groupby返回可迭代对象，可以使用for循环遍历：


```python
grouped = df.groupby('Year')
# 遍历每个分组
for year,group in grouped:
    print (year)
    print (group)

```

##### 获得一个分组细节

```python
grouped = df.groupby('Year')
print (grouped.get_group(2014))

```

##### 分组聚合

聚合函数为每个组返回聚合值。当创建了分组(*group by*)对象，就可以对每个分组数据执行求和、求标准差等操作。


```python
# 聚合每一年的平均的分
grouped = df.groupby('Year')
print (grouped['Points'].agg(np.mean))
# 聚合每一年的分数之和、平均分、标准差
grouped = df.groupby('Year')
agg = grouped['Points'].agg([np.sum, np.mean, np.std])
print (agg)

```