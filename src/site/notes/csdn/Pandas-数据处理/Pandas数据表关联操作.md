---
{"dg-publish":true,"permalink":"/csdn/Pandas-数据处理/Pandas数据表关联操作/","title":"Pandas数据表关联操作","tags":["python","pandas","数据分析"],"dg-note-properties":{"category":"Pandas / 数据处理","title":"Pandas数据表关联操作","source":"csdn","created":"2021-08-11","tags":["python","pandas","数据分析"],"url":"https://blog.csdn.net/weixin_45536921/article/details/119599383"}}
---



#### pandas数据表关联操作

Pandas具有功能全面的高性能内存中连接操作，与SQL等关系数据库非常相似。

Pandas提供了一个单独的```
merge()
```函数，作为DataFrame对象之间所有标准数据库连接操作的入口。


**合并两个DataFrame：**


```python
import pandas as pd
left = pd.DataFrame({
         'student_id':[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20],
         'student_name': ['Alex', 'Amy', 'Allen', 'Alice', 'Ayoung', 'Billy', 'Brian', 'Bran', 'Bryce', 'Betty', 'Emma', 'Marry', 'Allen', 'Jean', 'Rose', 'David', 'Tom', 'Jack', 'Daniel', 'Andrew'],
         'class_id':[1,1,1,2,2,2,3,3,3,4,1,1,1,2,2,2,3,3,3,2], 
         'gender':['M', 'M', 'F', 'F', 'M', 'M', 'F', 'F', 'M', 'M', 'F', 'F', 'M', 'M', 'F', 'F', 'M', 'M', 'F', 'F'], 
         'age':[20,21,22,20,21,22,23,20,21,22,20,21,22,23,20,21,22,20,21,22], 
         'score':[98,74,67,38,65,29,32,34,85,64,52,38,26,89,68,46,32,78,79,87]})
right = pd.DataFrame(
         {'class_id':[1,2,3,5],
         'class_name': ['ClassA', 'ClassB', 'ClassC', 'ClassE']})
# 合并两个DataFrame
data = pd.merge(left,right)
print(data)

```
**使用“how”参数合并DataFrame：**


```python
# 合并两个DataFrame (左连接)
rs = pd.merge(left, right, how='left')
print(rs)

```
其他合并方法同数据库相同：


合并方法SQL等效描述```
left
``````
LEFT OUTER JOIN
```使用左侧对象的键```
right
``````
RIGHT OUTER JOIN
```使用右侧对象的键```
outer
``````
FULL OUTER JOIN
```使用键的联合```
inner
``````
INNER JOIN
```使用键的交集试验：


```python
# 合并两个DataFrame (左连接)
rs = pd.merge(left,right,on='subject_id', how='right')
print(rs)
# 合并两个DataFrame (左连接)
rs = pd.merge(left,right,on='subject_id', how='outer')
print(rs)
# 合并两个DataFrame (左连接)
rs = pd.merge(left,right,on='subject_id', how='inner')
print(rs)

```

#### pandas透视表与交叉表

有如下数据：


```python
import pandas as pd
left = pd.DataFrame({
         'student_id':[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20],
         'student_name': ['Alex', 'Amy', 'Allen', 'Alice', 'Ayoung', 'Billy', 'Brian', 'Bran', 'Bryce', 'Betty', 'Emma', 'Marry', 'Allen', 'Jean', 'Rose', 'David', 'Tom', 'Jack', 'Daniel', 'Andrew'],
         'class_id':[1,1,1,2,2,2,3,3,3,4,1,1,1,2,2,2,3,3,3,2], 
         'gender':['M', 'M', 'F', 'F', 'M', 'M', 'F', 'F', 'M', 'M', 'F', 'F', 'M', 'M', 'F', 'F', 'M', 'M', 'F', 'F'], 
         'age':[20,21,22,20,21,22,23,20,21,22,20,21,22,23,20,21,22,20,21,22], 
         'score':[98,74,67,38,65,29,32,34,85,64,52,38,26,89,68,46,32,78,79,87]})
right = pd.DataFrame(
         {'class_id':[1,2,3,5],
         'class_name': ['ClassA', 'ClassB', 'ClassC', 'ClassE']})
# 合并两个DataFrame
data = pd.merge(left,right)
print(data)

```
**透视表**


透视表(pivot table)是各种电子表格程序和其他数据分析软件中一种常见的数据汇总工具。**它根据一个或多个键对数据进行分组聚合，并根据每个分组进行数据汇总**。


```python
# 以class_id与gender做分组汇总数据，默认聚合统计所有列
print(data.pivot_table(index=['class_id', 'gender']))

# 以class_id与gender做分组汇总数据，聚合统计score列
print(data.pivot_table(index=['class_id', 'gender'], values=['score']))

# 以class_id与gender做分组汇总数据，聚合统计score列，针对age的每个值列级分组统计
print(data.pivot_table(index=['class_id', 'gender'], values=['score'], 
                       columns=['age']))

# 以class_id与gender做分组汇总数据，聚合统计score列，针对age的每个值列级分组统计，添加行、列小计
print(data.pivot_table(index=['class_id', 'gender'], values=['score'], 
                       columns=['age'], margins=True))

# 以class_id与gender做分组汇总数据，聚合统计score列，针对age的每个值列级分组统计，添加行、列小计
print(data.pivot_table(index=['class_id', 'gender'], values=['score'], 
                       columns=['age'], margins=True, aggfunc='max'))

```
**交叉表**


交叉表(cross-tabulation, 简称crosstab)是一种用于**计算分组频率的特殊透视表**：


```python
# 按照class_id分组，针对不同的gender，统计数量
print(pd.crosstab(data.class_id, data.gender, margins=True))

```