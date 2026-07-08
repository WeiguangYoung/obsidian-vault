---
{"dg-publish":true,"permalink":"/csdn/Pandas-数据处理/Pandas可视化与文件读取/","title":"Pandas可视化与文件读取","tags":["python","可视化","pandas"],"dg-note-properties":{"category":"Pandas / 数据处理","title":"Pandas可视化与文件读取","source":"csdn","created":"2021-08-11","tags":["python","可视化","pandas"],"url":"https://blog.csdn.net/weixin_45536921/article/details/119599462"}}
---



#### pandas可视化

**基本绘图：绘图**


```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as mp 

df = pd.DataFrame(np.random.randn(10,4),index=pd.date_range('2018/12/18',
   periods=10), columns=list('ABCD'))
df.plot()
mp.show()

```
plot方法允许除默认线图之外的少数绘图样式。 这些方法可以作为```
plot()
```的```
kind
```关键字参数。这些包括 ：


- ```
bar
```或```
barh
```为条形
- ```
hist
```为直方图
- ```
scatter
```为散点图

**条形图**


```python
df = pd.DataFrame(np.random.rand(10,4),columns=['a','b','c','d'])
df.plot.bar()
# df.plot.bar(stacked=True)
mp.show()

```
**直方图**


```python
df = pd.DataFrame()
df['a'] = pd.Series(np.random.normal(0, 1, 1000)-1)
df['b'] = pd.Series(np.random.normal(0, 1, 1000))
df['c'] = pd.Series(np.random.normal(0, 1, 1000)+1)
print(df)
df.plot.hist(bins=20)
mp.show()

```
**散点图**


```python
df = pd.DataFrame(np.random.rand(50, 4), columns=['a', 'b', 'c', 'd'])
df.plot.scatter(x='a', y='b')
mp.show()

```
**饼状图**


```python
df = pd.DataFrame(3 * np.random.rand(4), index=['a', 'b', 'c', 'd'], columns=['x'])
df.plot.pie(subplots=True)
mp.show()

```

#### 数据读取与存储

**读取与存储csv：**


```python
# filepath 文件路径。该字符串可以是一个URL。有效的URL方案包括http，ftp和file 
# sep 分隔符。read_csv默认为“,”，read_table默认为制表符“[Tab]”。
# header 接收int或sequence。表示将某行数据作为列名。默认为infer，表示自动识别。
# names 接收array。表示列名。
# index_col 表示索引列的位置，取值为sequence则代表多重索引。 
# dtype 代表写入的数据类型（列名为key，数据格式为values）。
# engine 接收c或者python。代表数据解析引擎。默认为c。
# nrows 接收int。表示读取前n行。

pd.read_table(
    filepath_or_buffer, sep='\t', header='infer', names=None, 
    index_col=None, dtype=None, engine=None, nrows=None) 
pd.read_csv(
    filepath_or_buffer, sep=',', header='infer', names=None, 
    index_col=None, dtype=None, engine=None, nrows=None)

```
```python
DataFrame.to_csv(excel_writer=None, sheetname=None, header=True, index=True, index_label=None, mode=’w’, encoding=None) 

```
**读取与存储excel：**


```python
# io 表示文件路径。
# sheetname 代表excel表内数据的分表位置。默认为0。 
# header 接收int或sequence。表示将某行数据作为列名。默认为infer，表示自动识别。
# names 表示索引列的位置，取值为sequence则代表多重索引。
# index_col 表示索引列的位置，取值为sequence则代表多重索引。
# dtype 接收dict。数据类型。
pandas.read_excel(io, sheetname=0, header=0, index_col=None, names=None, dtype=None)

```
```python
DataFrame.to_excel(excel_writer=None, sheetname=None, header=True, index=True, index_label=None, mode=’w’, encoding=None) 

```
**读取与存储JSON：**


```python
# 通过json模块转换为字典，再转换为DataFrame
pd.read_json('../ratings.json')

```