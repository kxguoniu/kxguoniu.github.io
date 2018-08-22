---
layout: post
title: Python压缩文件
categories: Python
description: python实现压缩文件的体会
keywords: Python
---

## 环境

1.Linux系统

2.python2.7

3.zipfile模块

## 压缩文件

```python
def zip_dir(dirname=DIRNAME, zipfilename=ZIPFILENAME):
    '''
    压缩文件
    :dirname: 压缩文件夹
    :zipfilename: 压缩后文件名
    '''
    if not os.path.isdir(dirname):
        return False
    filelist = [dirname]
    s_time = read_time()
    print '上次结束时间',s_time
    with zipfile.ZipFile(zipfilename, 'w', zipfile.zlib.DEFLATED) as fp:
        #循环深度优先搜索(非递归)
        while len(filelist) > 0:
            dirpath = filelist[-1]
            pathlist = []
            for file in os.listdir(dirpath):
                filepath = os.path.join(dirpath, file)
                file = filepath[len(dirname)+1:]
                if os.path.isfile(filepath) and searchfile(file, s_time):
                    fp.write(filepath, file)
                    print '压缩文件',file, 'Pass'
                elif os.path.isdir(filepath):
                    fp.write(filepath, file)
                    pathlist.append(filepath)
            filelist.pop(-1)
            pathlist.reverse()
            filelist.extend(pathlist)
    return True
```

# 解压文件

```python
def unzip_dir(zipfilename=ZIPFILENAME, unziptodir=UNZIPTODIR):
    '''
    解压缩
    ;zipfilename: 压缩包路径
    ;unziptodir: 解压路径
    '''
    if not os.path.exists(unziptodir):
        os.mkdir(unziptodir)
    with zipfile.ZipFile(zipfilename) as fp:
        for name in fp.namelist():
            filepath = os.path.join(unziptodir, name)
            if name.endswith('/'):
                if not os.path.isdir(filepath):
                    os.mkdir(filepath)
            else:
                dirpath = os.path.dirname(filepath)
                if not os.path.exists(dirpath):
                    os.mkdir(dirpath)
                fp.extract(name, unziptodir)
```

# 筛选文件

```python
def searchfile(filepath, s_time):
    '''
    筛选文件
    :filepath: 文件相对路径
    :s_time: 上次压缩时间
    '''
    print '匹配文件',filepath
    result = re.search(PATTERN, filepath)
    if not s_time:
        return True
    if result is not None:
        time = result.group(1)
        e_time = datetime.strptime(time, '%Y-%m-%d')
        if (e_time - s_time).days > 0:
            return True
    return False
```

# 读取和保存记录

```python
def read_time():
    '''
    读取上次压缩时间
    '''
    if not os.path.isfile(LOGFILE):
        return False
    with open(LOGFILE, 'r+') as file:
        time = file.read()
        s_time = datetime.strptime(time, '%Y-%m-%d')
    return s_time

def save_time():
    '''
    保存本次压缩时间
    '''
    time = datetime.now().strftime('%Y-%m-%d')
    with open(LOGFILE, 'w') as file:
        file.write(time)
```
