---
title: Python Progress Bar
date: 2024-09-20T17:28:51+08:00
tags:
   - tqdm
   - python 
   - program language
---

在使用`python`编写程序时，尤其是命令行程序，很多时候需要执行一些耗时较长的任务，比如爬虫任务、机器学习领域对大数据集做遍历处理等场景。**进度条（Progress Bar）** 可以展示当前任务是否已经完成、完成的百分比，甚至能够告诉你还需要等待多久。`python`内置提供了 [tqdm](https://github.com/tqdm/tqdm) 模块，通过简单几行代码就可以完成进度条的打印。

官方提供了一个最简单的示例：

```
>>> import tqdm
>>> for i in tqdm.trange(100):
...     pass
...
100%|██████████████████████████| 100/100 [00:00<00:00, 921825.05it/s]
```

### 丰富你的进度条

#### 描述进度条

你可以通过实例化tqdm时指定`desc`属性来为进度条提供一个简介的描述性信息。

> `tqdm.tqdm(total=100, desc='Users')`

完整的示例代码如下：

```
import tqdm
import time

pbar = tqdm.tqdm(total=100, desc='Users')
for i in range(100):
    time.sleep(0.01)  # 模拟任务处理消耗的时间
    pbar.update()
pbar.close()
```

代码输出效果为：

```
Users:  38%|████████████▉                     | 38/100 [00:00<00:00, 94.51it/s]
```

#### 使描述信息动起来

通过在实例化tqdm时指定`desc`属性，可以在进度条前方展示描述信息，如果需要在执行过程中动态调整这个内容，可以通过调用tqdm的`set_description`函数实现。

> `pbar = tqdm.tqdm(total=100, desc='Users Na/Na')`  
> `pbar.set_description(f'Users {current_idx}/{total}')`

完整的示例代码如下：

```
import tqdm
import time

total = 100
pbar = tqdm.tqdm(total=total, desc='Users Na/Na')
for i in range(total):
    time.sleep(0.01)
    pbar.set_description(f'Users {i+1}/{total}')
    pbar.update()
pbar.close()
```

代码输出效果为：

```
Users 58/100:  58%|███████████████▋           | 58/100 [00:00<00:00, 92.15it/s]
```

#### 多个进度条

通过构建两个tqdm实例，为它们赋予不同的的`position`属性，可以达到输出多个进度条的效果。

> `tqdm.tqdm(total=100, position=0)`  
> `tqdm.tqdm(total=100, position=1)`

完整的示例代码如下：

```
import tqdm
import time

outer_pbar = tqdm.tqdm(total=100, desc='Users', position=0)
inner_pbar = tqdm.tqdm(total=50, desc='Behavior Analysis', position=1)
for i in range(100):
    for j in range(50):
        time.sleep(0.01)  # 模拟用户行为分析消耗的时间
        inner_pbar.update()
    inner_pbar.reset()
    outer_pbar.update()
inner_pbar.close()
outer_pbar.close()
```

代码输出效果为：

```
Users:  17%|█████▊                            | 17/100 [00:09<00:44,  1.86it/s]
Behavior Analysis:  60%|█████████████▊         | 30/50 [00:00<00:00, 93.12it/s]
```

#### 在进度条上方打印日志

如果你希望在更新进度条的时候，打印一些日志，直接使用`print()`是不可以的，因为tqdm对于`\r`的处理，会使你的输出内容错乱。tqdm提供了`write`函数来替代`print`。

> `tqdm.tqdm(total=100, desc='Users')`  
> `tqdm.write('msg...')`  

完整的示例代码如下：

```
import tqdm
import time

pbar = tqdm.tqdm(total=100, desc='Users', position=0)
for i in range(100):
    time.sleep(0.01)
    pbar.write(f'User {i}')  # 使用write()替换print()
    # print(f'User {i}')  # 使用print()会使控制台输出错乱
    pbar.update()
pbar.close()
```

代码输出效果为：

```
>>> OUTPUT <<<
...
User 98
User 99
Users: 100%|█████████████████████████████████| 100/100 [00:01<00:00, 92.27it/s]
>>> OUTPUT <<<
```

#### 为进度条增加状态栏

简单的展示进度条，还是会缺失很多数据，而且tqdm提供的`desc`属性，由于和进度条展示在同一行，受制于长度限制，并不是所有信息都适合放在这里。

举例来说，在一个批量处理文件的脚本中，我们希望在控制台输出如下信息：

```
Progress:  70%|███████████████████████▊          | 70/100 [00:00<00:00, 95.24it/s]
Current File: Name[a.txt] Size[11.5MB]
```

需要说明的是，tqdm本身是不支持在进度条下方输出内容的，但是我们可以通过创建两个tqdm实例来实现功能，第一个是真实的进度条，第二个是假的进度条，它的作用仅用来展示信息。

> 1. 定义第一个tqdm实例用于进度条展示  
> `p_bar = tqdm.tqdm(total=100, desc='Progress', position=0)`
> 2. 定义第二个tqdm实例，仅渲染其desc信息  
> `log_bar = tqdm.tqdm(total=0, position=1, bar_format='{desc}')`
> 3. 更新第二个tqdm实例的desc  
> `log_bar.set_description_str(f'Current File: Name[{file_name}] Size[{file_size}MB]]')`  
> 这里我们使用的是`set_description_str`而不是`set_description`函数，是因为`set_description_str`函数可以避免在字符串后增加[冒号`:`]


完整的示例代码如下：

```
import tqdm
import time

p_bar = tqdm.tqdm(total=100, desc='Progress', position=0)  # 第一个tqdm实例，作为进度条
log_bar = tqdm.tqdm(total=0, position=1, bar_format='{desc}')  # 第二个tqdm实例，仅承载状态数据，totol配置为0
for i in range(100):
    time.sleep(0.01)
    file_name = f'f_{i}.txt' # 模拟文件名
    file_size = f'{i}' # 模拟文件大小
    p_bar.update() # 更新进度条
    log_bar.set_description_str(f'Current File: Name[{file_name}] Size[{file_size}MB]]') # 更新文件信息
p_bar.close()
log_bar.close()
```

代码输出效果为：

```
Progress:  50%|███████████████▌               | 50/100 [00:00<00:00, 92.02it/s]
Current File: Name[f_49.txt] Size[49MB]]
```

> 
