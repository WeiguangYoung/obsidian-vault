---
{"dg-publish":true,"permalink":"/csdn/Pandas-数据处理/openpyxl使用/","title":"openpyxl使用","tags":["excel"],"dg-note-properties":{"category":"Pandas / 数据处理","title":"openpyxl使用","source":"csdn","created":"2021-06-17","tags":["excel"],"url":"https://blog.csdn.net/weixin_45536921/article/details/117985325"}}
---



#### openpyxl使用

openpyxl官方文档

https://openpyxl.readthedocs.io/en/stable/index.html


使用阿里镜像源安装依赖


```shell
pip3 install openpyxl -i https://mirrors.aliyun.com/pypi/simple/

```

##### 创建Workbook和Sheet

Workbook就是一个excel文件的python对像


```python
from openpyxl import Workbook, load_workbook
# Create a workbook
wb = Workbook()

ws = wb.active
ws.title = "MySheet1"
ws.sheet_properties.tabColor = "1072BA"

ws1 = wb["MySheet1"]  # 获取指定sheet
ws2 = wb.create_sheet("MySheet2", index=1)  # 指定插入第几个sheet，0为第一个，1为第二个
ws3 = wb.create_sheet("MySheet3", )  # 默认插入最后一个
wb.copy_worksheet(ws3)  # 复制MySheet3到MySheet3 Copy

# 打印所有的sheet名称
print(wb.sheetnames)

```

##### 访问cell

读写cell，接上


```python
# Accessing one cell
ws["A1"] = 4
d = ws.cell(row=2, column=2, value=10)

# Accessing many cells
cell_range = ws['A1':'B2']
colC = ws['B']
col_range = ws['A:B']
row10 = ws[2]
row_range = ws[1:2]

# 指定区域获取行
for row in ws.iter_rows(min_row=1, max_col=2, max_row=2):
    print(row)

# 指定区域获取列
for row in ws.iter_cols(min_row=1, max_col=2, max_row=2):
    print(row)

# 获取全部行列
# print(tuple(ws.rows))
# print(tuple(ws.columns))

# Values
for row in ws.values:
    print(row)

cell_B1 = ws["B1"]
cell_B1.value = 6

# 使用公式 
ws["A2"] = "=SUM(A1, B1)"

# values_only 指定读取值
for row in ws.iter_rows(min_row=1, max_col=2, max_row=2, values_only=True):
    print(row)

```

##### 保存文件

```python
wb.save("01_mybook.xlsx")

```

##### 读取文件

```python
wb2 = load_workbook("01_mybook.xlsx")
print(wb2.sheetnames)

```