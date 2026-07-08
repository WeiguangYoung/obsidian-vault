---
{"dg-publish":true,"permalink":"/csdn/Pandas-数据处理/Pandas时间日期处理/","title":"Pandas时间日期处理","tags":["python","pandas"],"dg-note-properties":{"category":"Pandas / 数据处理","title":"Pandas时间日期处理","source":"csdn","created":"2021-08-11","tags":["python","pandas"],"url":"https://blog.csdn.net/weixin_45536921/article/details/119599174"}}
---



**pandas时间日期处理**


```python
# pandas识别的日期字符串格式
dates = pd.Series(['2011', '2011-02', '2011-03-01', '2011/04/01', 
                   '2011/05/01 01:01:01', '01 Jun 2011'])
# to_datetime() 转换日期数据类型
dates = pd.to_datetime(dates)
print(dates, dates.dtype, type(dates))
# datetime类型数据支持日期运算
delta = dates - pd.to_datetime('1970-01-01')
# 获取天数数值
print(delta.dt.days)

```
Series.dt提供了很多日期相关操作，如下：


```python
Series.dt.year	The year of the datetime.
Series.dt.month	The month as January=1, December=12.
Series.dt.day	The days of the datetime.
Series.dt.hour	The hours of the datetime.
Series.dt.minute	The minutes of the datetime.
Series.dt.second	The seconds of the datetime.
Series.dt.microsecond	The microseconds of the datetime.
Series.dt.week	The week ordinal of the year.
Series.dt.weekofyear	The week ordinal of the year.
Series.dt.dayofweek	The day of the week with Monday=0, Sunday=6.
Series.dt.weekday	The day of the week with Monday=0, Sunday=6.
Series.dt.dayofyear	The ordinal day of the year.
Series.dt.quarter	The quarter of the date.
Series.dt.is_month_start	Indicates whether the date is the first day of the month.
Series.dt.is_month_end	Indicates whether the date is the last day of the month.
Series.dt.is_quarter_start	Indicator for whether the date is the first day of a quarter.
Series.dt.is_quarter_end	Indicator for whether the date is the last day of a quarter.
Series.dt.is_year_start	Indicate whether the date is the first day of a year.
Series.dt.is_year_end	Indicate whether the date is the last day of the year.
Series.dt.is_leap_year	Boolean indicator if the date belongs to a leap year.
Series.dt.days_in_month	The number of days in the month.

```

##### DateTimeIndex

通过指定周期和频率，使用```
date.range()
```函数就可以创建日期序列。 默认情况下，范围的频率是天。


```python
import pandas as pd
# 以日为频率
datelist = pd.date_range('2019/08/21', periods=5)
print(datelist)
# 以月为频率
datelist = pd.date_range('2019/08/21', periods=5,freq='M')
print(datelist)
# 构建某个区间的时间序列
start = pd.datetime(2017, 11, 1)
end = pd.datetime(2017, 11, 5)
dates = pd.date_range(start, end)
print(dates)

```
```
bdate_range()
```用来表示商业日期范围，不同于```
date_range()
```，它不包括星期六和星期天。


```python
import pandas as pd
datelist = pd.bdate_range('2011/11/03', periods=5)
print(datelist)

```