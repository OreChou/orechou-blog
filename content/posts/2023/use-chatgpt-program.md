---
title: 【AI 赋能小日常】使用 ChatGPT 辅助编程
date: 2023-06-14 18:30:00
tags: [AI,ChatGPT]
---

最近半导体和 AI 概念股狂涨，在二级市场没有搭上这趟车的我坐不住了。不敢追高的我只能写一写博客来缓解郁闷的心情。这篇文章是我作为程序员，介绍一下日常工作里，如何使用 ChatGPT 简化我的工作。

## 分析需求
比如本周有一个需求：对含有文件下载 URL 链接的 Excel 文件进行处理，将每一行中的链接的文件下载下来后进行打包，打包后的文件名使用该行的某几个属性值命名。
举一个例子，比如 Excel 的文件如下

| 表头1 | 表头2  | 表头3  | 表头4 |
| ----- | --- | --- | --- |
| A |  B   |   https://www.apple.com.cn/privacy/docs/A_Day_in_the_Life_of_Your_Data.pdf  | https://www.apple.com.cn/privacy/docs/A_Day_in_the_Life_of_Your_Data.pdf |

需要将表头 3 和表头 4 的 pdf 下载下来，并打包成一个压缩包，压缩包以表头 1 和表头 2 命名，例如第一行就是 A_B.zip

## ChatGPT 的交互

### 回合 1 定义角色
首先给 ChatGPT 一个角色，Prompt 如下：
> 我希望你能扮演一个软件开发者的角色。我将提供一些关于网络应用需求的具体信息，而你的工作是提出一个架构和代码，而你的技术栈是无所不能的。

ChatGPT 会欣然答应，立即进入角色：
> 非常好！请告诉我关于网络应用需求的具体信息，我会根据这些信息来设计一个合适的架构和代码。


### 回合 2 提交需求
我将需求描述调整一下，以写程序的角度告诉他应该完成什么内容，Prompt 如下：
> 我有一个需求：对含有文件 URL 下载链接的 Excel 文件进行处理，将每一行中的链接的文件下载下来后进行打包成 zip 文件，打包后的文件名使用该行的某几个属性值命名。注意这里每一行最终都是不同的 ZIP 文件。
> 我要你设计一个方法完成这个功能，方法的入参为 excel 的文件路径，下载链接所在的列索引列表，命名所需要指定的列索引列表，压缩包的存储路径（非必传，不传的话在执行路径下生成）。 你将使用 Python3 来实现这个需求功能。

ChatGPT 会很快给出相关的代码及说明：

```python
import os
import requests
import pandas as pd
from zipfile import ZipFile


def process_excel(excel_path, link_columns, name_columns, zip_dir=None):
    df = pd.read_excel(excel_path)

    # Use current directory if zip_dir is not specified
    if zip_dir is None:
        zip_dir = os.getcwd()

    for index, row in df.iterrows():
        # Create a directory for each row to store downloaded files
        row_dir = os.path.join(zip_dir, f"row_{index}")
        os.makedirs(row_dir, exist_ok=True)

        # Download each file in link_columns
        for col in link_columns:
            url = row[col]
            filename = os.path.join(row_dir, url.split("/")[-1])
            response = requests.get(url)
            with open(filename, 'wb') as f:
                f.write(response.content)

        # Create a zip file for each row
        zip_filename = "_".join([str(row[col]) for col in name_columns]) + ".zip"
        zip_path = os.path.join(zip_dir, zip_filename)
        with ZipFile(zip_path, 'w') as zipf:
            for foldername, subfolders, filenames in os.walk(row_dir):
                for filename in filenames:
                    zipf.write(os.path.join(foldername, filename), arcname = filename)

        # Clean up downloaded files
        for filename in os.listdir(row_dir):
            os.remove(os.path.join(row_dir, filename))
        os.rmdir(row_dir)
```

> 这是一个相对复杂的任务，以下是我设计的一个函数以完成此任务。在这个函数中，我使用了以下Python库：
> - pandas：用于读取和操作Excel数据
> - requests：用于下载文件
> - zipfile：用于创建zip文件
> - os：用于处理文件和目录路径

> 这个函数首先读取Excel文件并逐行处理。对于每一行，它首先下载链接列中的每个文件，然后使用name_columns中的列值创建一个ZIP文件。下载的文件会暂存在一个临时文件夹中，这个文件夹在ZIP文件创建完成后会被删除。
注意，这个函数没有处理可能出现的错误，例如链接无效，无法下载文件，没有写入权限等。在实际使用中，你可能需要添加适当的错误处理。


### 回合 3 改进优化
简单 review 了下 ChatGPT 给出的代码，是能 work 的。但考虑到文件下载是个比较耗时的操作，并且如何这个 Excel 如果行数较多的话，处理完整个 Excel 的话可能耗时比较久。所以我们可以改进一下，让他提供一个多线程的版本来加快处理的速度。

给出的 Prompt 如下：
```
考虑到 Excel 的行数可能比较多，处理完比较耗时。我需要将上面的方法改成一个多线程的版本，入参新增一个指定的最大线程数，然后每一行分配到线程池中提交执行。
```

ChatGPT 立刻给出了多线程的版本：
```python
import os
import requests
import pandas as pd
from zipfile import ZipFile
from concurrent.futures import ThreadPoolExecutor

def download_and_zip(row, link_columns, name_columns, zip_dir):
    # Create a directory for each row to store downloaded files
    row_dir = os.path.join(zip_dir, f"row_{row.name}")
    os.makedirs(row_dir, exist_ok=True)

    # Download each file in link_columns
    for col in link_columns:
        url = row[col]
        filename = os.path.join(row_dir, url.split("/")[-1])
        response = requests.get(url)
        with open(filename, 'wb') as f:
            f.write(response.content)

    # Create a zip file for each row
    zip_filename = "_".join([str(row[col]) for col in name_columns]) + ".zip"
    zip_path = os.path.join(zip_dir, zip_filename)
    with ZipFile(zip_path, 'w') as zipf:
        for foldername, subfolders, filenames in os.walk(row_dir):
            for filename in filenames:
                zipf.write(os.path.join(foldername, filename), arcname = filename)

    # Clean up downloaded files
    for filename in os.listdir(row_dir):
        os.remove(os.path.join(row_dir, filename))
    os.rmdir(row_dir)


def process_excel_threaded(excel_path, link_columns, name_columns, max_threads, zip_dir=None):
    df = pd.read_excel(excel_path)

    # Use current directory if zip_dir is not specified
    if zip_dir is None:
        zip_dir = os.getcwd()

    with ThreadPoolExecutor(max_workers=max_threads) as executor:
        futures = [executor.submit(download_and_zip, row, link_columns, name_columns, zip_dir) for _, row in df.iterrows()]
        for future in futures:
            future.result()

```


> 我们可以使用Python的concurrent.futures模块中的ThreadPoolExecutor类来实现多线程。
> 这个版本的函数创建了一个线程池，并且为每一行创建了一个新的任务来下载文件和创建zip文件。注意，我们创建的任务数应该等于Excel文件中的行数，而不是我们设定的最大线程数。ThreadPoolExecutor类会自动管理线程池，并确保任何时候都只有max_threads数量的线程在运行。请注意，这个版本的函数同样没有处理可能出现的错误。在实际使用中，你可能需要添加适当的错误处理。

到现在 ChatGPT 已经给出功能性比较完备的代码了，添加一些日志和异常处理后，直接使用并没有什么问题。总的来说，如果让我自己来写这个脚本，我可能需要一两个小时来完成，而 ChatGPT 只用了几分钟就完成了。节省下来的时间可以去完成更有价值的事情，这就是 AI 赋能的价值所在。