---
layout: post
title: "使用 Python 脚本删除重复的图片"
description: "使用 Python 脚本删除重复的图片"
date: 2023-05-03
tags: [python, picture]
---

由于各种原因导致电脑上备份的照片有大量的重复，因此希望能够删除这些重复的照片。

这些重复的照片是由于云同步，换手机同步等原因导致的重复。
所以，这些重复的照片的文件内容的二进制数据是完全一样的，而不是视觉上的一样。
因此可以通过简单的计算文件的 MD5 来判断文件是否重复。

下面简短的代码实现删除重复的照片。
这里只是简单的删除重复，而没有办法选择保留哪些路径下的照片。
比如，我们原来有些照片已经是经过分类了，这些目录下的照片是要保留的，因此就不应该删除这些目录下的照片。

```python
import os
import hashlib
import sys

IMAGE_EXTENSIONS = ['.jpg', '.jpeg', '.png', '.bmp', '.gif']
md52path = {}

def calculate_md5_for_file(file_path):
    with open(file_path, 'rb') as f:
        content = f.read()
        md5_hash = hashlib.md5()
        md5_hash.update(content)
        return md5_hash.hexdigest()

def is_image_file(file_path):
    _, extension = os.path.splitext(file_path)
    return extension.lower() in IMAGE_EXTENSIONS

def calculate_md5_for_dir(dir_path):
    global md52path
    for dirpath, dirnames, filenames in os.walk(dir_path):
        for file_name in filenames:
            if not is_image_file(file_name):
                continue

            file_path = os.path.join(dirpath, file_name)
            md5_hash = calculate_md5_for_file(file_path)
            if not md5_hash in md52path:
                md52path[md5_hash] = file_path
            else:
                print(file_path, md52path[md5_hash], sep=",")
                os.remove(file_path)

for dir in sys.argv[1:]:
    calculate_md5_for_dir(dir)
```

把上面的代码保存到文件 remove_duplicate.py 中。
然后通过执行 `python path_to/remove_duplicate.py  dir1 dir2 ...` 这样可以删除多个目录下的重复文件。
