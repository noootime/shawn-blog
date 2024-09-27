---
title: Pandas Sql Operation
date: 2024-09-27T10:31:28+08:00
tags: 
    - blogumentation
    - python
    - pandas
slug: pandas-sql-operation
---

最近需要基于mysql数据库的数据做一些处理，由于涉及多个表，且每张表的数据量都比较大，索引设计的也不是很好，导致很多表关联查询非常慢。由于pandas提供了很强大的数据分析能力，而且支持很多复杂的关联性操作，因此可以考虑将需要的表都从数据库查询出来，落入本地文件后，在本地通过pandas加载入内存进行处理，只要内存足够，还是可以满足分析需求的。

我的处理方式分为三部分：导出数据，加载数据，分析数据，这里主要记录一下导出数据和加载数据的一些关键代码片段，方便将来快速回顾。

最基础的从数据库读取数据并写入DataFrame的方式如下，你需要有一个数据库连接，使用`mysqlconnector`或者`sqlalchemy`都可以，因为我在查资料时发现`sqlalchemy`好像比`mysqlconnector`更常用，我就使用它来试一试。

```python
import pandas as pd
from sqlalchemy import create_engine

user = ''
password = ''
engine = create_engine(f"mysql+mysqlconnector://{user}:{password}@127.0.0.1", echo=True)

sql = "SELECT * FROM table_name WHERE ..."
df = pd.read_sql(sql, con=engine)
df.to_csv('xxx/yyy/file_name.csv')
```

如果表的数据量很大，那么需要分页查询，并且最好的方式应该是每查出一批数据，就写一次文件。如果你等所有数据全部分页查询完成，再一次性写入文件，可能会遇到一些令人崩溃的问题：

- 数据量太大，超过内存，你好不容易花了一个多小时，数据快跑完了，内存不够，程序崩溃了
- 你的try catch没有做好，一旦网络不好访问数据库失败，也会碰到和上面一样恶心的问题
- 即使你的内存足够，网络够好，整个过程没有什么问题，但是导出的文件可能非常大，这对你的文件传输、加载也非常不好

因此，我通过将数据分批写入文件的方式，并且采用`parquet`文件类型，而不是`csv`，因为这个格式压缩率也会好一些

```python
import pandas as pd
from sqlalchemy import create_engine

user = ''
password = ''
engine = create_engine(f"mysql+mysqlconnector://{user}:{password}@127.0.0.1", echo=True)

offset = 0
page_size = 10

chunk_no = 1
chunk = []
# 每个块中数据量为1w
chunk_data_capacity = 10000
# 每个块的页数量
chunk_capacity = chunk_data_capacity / page_size

while True:
    query = f"SELECT * FROM table_name WHERE id > {offset} LIMIT {page_size}"
    query_df = pd.read_sql(sql=query, con=engine)
    if query_df.empty:
        break
    chunk.append(query_df)
    if len(chunk) >= chunk_capacity:
        # chunk中数据满了，合并 -> 存储 -> 清空
        chunk_df = pd.concat(chunk, ignore_index=True)
        chunk_df.to_parquet(f'fila_name-chunk{chunk_no}.parquet')
        chunk = []
        chunk_no += 1
    offset = query_df['id'].max()

df = pd.read_sql(sql, con=engine)
df.to_csv('xxx/yyy/file_name.csv')
```

如何将分块的数据合并加载: 

```python
import os
import pandas as pd

directory = 'xxx'
def loads_data():
    dfs = []
    for filename in os.listdir(directory):
        if filename.startswith('fila_name-chunk') and filename.endswith('.parquet'):
            file_path = os.path.join(directory, filename)
            df = pd.read_parquet(file_path)
            dfs.append(df)
    df = pd.concat(dfs, ignore_index=True)
    return df
```

