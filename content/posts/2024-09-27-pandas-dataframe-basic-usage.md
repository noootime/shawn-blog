---
title: Pandas-DataFrame Basic Usage
date: 2024-09-27T12:03:55+08:00
tags:
    - blogumentation
    - python
    - pandas
slug: pandas-dataframe-basic-usage
draft: true
---

重命名列名

```python
df.rename(columns = {
    'col_1': '1_col',
    'col_2': '2_col',
})
```

根据列名提取指定列

```python
# 仅提取1列
df['col_1']
# 提取超过1列
df[['col_1', 'col_2']]
```

根据条件筛选数据

```python
# 筛选出所有col_1列内容为空的数据
df[df['col_1'].isna()]

# 筛选出所有col_1列内容不为空的数据，以下两种方式均可
df[~df['col_1'].isna()]
df[df['col_1'].notna()]

# 筛选出所有col_1列内容为"hello world"的数据
df[df['col_1'] == 'hello world']

# 筛选出所有col_1列内容为"hello"的数据 或者 col_2列内容为"python"的数据
df[(df['col_1'] == 'hello') | (df['col_2'] == 'python')]

# 筛选出所有col_1列内容为"hello"的数据 并且 col_2列内容为"python"的数据
df[(df['col_1'] == 'hello') & (df['col_2'] == 'python')]
```

数据去重

```python
# 去除重复数据
df.drop_duplicates()

# 根据col_1列进行去重
df.drop_duplicates(subset=['col_1'])
```

数据合并，假设有两个`DataFrame`，`df1`和`df2`，需要对它们做类似数据库的`join`操作

```python
# 类似数据库的left join，以两个DataFrame的id列作为匹配条件
df1.merge(df2, on='id', how='left')
# 如果有多个匹配条件
df1.merge(df2, on=['col_1', 'col_2'], how='left')
# 如果两个DateFrame的匹配列名字不同
df1.merge(df2, left_on='df1_id', right_on='df2_id', how='left')
```