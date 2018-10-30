# Python：Python自动化上传OSS

## 简介
>最近在学习Python，为之庞大的第三方库感到震撼。今天分享一个Python 自动化脚本，功能是将H5静态资源上传到OSS，以方便实现CDN加速，我将其放在Jenkins自动发布中使用。该脚本不是我的原创，是前同事留下的脚本，希望对需要的小伙伴有所帮助。

## 安装阿里云第三方库
 ```pip install oss2``` 
![img-w500](/images/201810301004.png)

## 脚本说明
accesskey，accesspassword，bucketname，ossBucket 需要根据自己账户情况作出调整

## 脚本内容

```
#!/usr/bin/python
# -*- coding:utf8 -*-

import os
import sys
import oss2

# 阿里云相关的token
ossAuth = oss2.Auth('accesskey', 'accesspassword')
ossBucket = oss2.Bucket(ossAuth, 'oss-cn-hangzhou.aliyuncs.com', 'bucketname')

# 本地需要上传的文件或者目录，sys.argv: 实现从程序外部向程序传递参数。
pathfile = sys.argv[1]

# 判断是否输入目标目录(在oss中的目录，程序会自动创建)，如果没有输入目标目录，则直接上传文件到oss的根目录下
if len(sys.argv) == 3:
    ossDir = sys.argv[2] + "/"
else:
    ossDir = ""

ee = [1]
ee[0] = 1
# 最后一次上传到哪个文件，第一次上传请修改ee[0]=1
ff = '550.jpg'


# 定义是目录
def list(dir):
    fs = os.listdir(dir)
    for f in fs:
        file = dir + "/" + f;
        print("file is" + ":" + file)
        if os.path.isdir(file):
            list(file)
        else:
            uploadDir(file)


# 上传带目录的文件到OSS
def uploadDir(path_filename):
    print("------------------")
    print(path_filename)
    remoteName = ossDir + path_filename.split('//')[1]
    print("remoteName is" + ":" + remoteName)
    print('uploading..', path_filename, 'remoteName', remoteName)
    if (ee[0] == 0 and remoteName == ff):
        ee[0] = 1
    if 1 == ee[0]:
        result = ossBucket.put_object_from_file(remoteName, path_filename)
        print('http_status: {0}'.format(result.status))


# 上传文件到OSS
def uploadFile(filename):
    remoteName = ossDir + os.path.basename(filename)
    print("remoteName is" + ":" + remoteName)
    print('uploading..', filename, 'remoteName', remoteName)
    if (ee[0] == 0 and remoteName == ff):
        ee[0] = 1
    if 1 == ee[0]:
        result = ossBucket.put_object_from_file(remoteName, filename)
        print('http_status: {0}'.format(result.status))


##判断是文件还是目录
if os.path.isdir(pathfile):
    if pathfile.endswith('/'):
        pass
    else:
        pathfile += "/"
    print("it's a directory")
    list(pathfile)
elif os.path.isfile(pathfile):
    print("it's a normal file")
    uploadFile(pathfile)
else:
    print("it's a special file (socket, FIFO, device file)")

```
## 执行效果展示
![img-w500](/images/201810301047.png)


## 参考文档

+ python之sys模块详解：<https://www.cnblogs.com/cherishry/p/5725184.html>
+ Aliyun OSS SDK for Python：<https://aliyun-oss-python-sdk.readthedocs.io/en/stable>
+ aliyun-oss-python-sdk Github地址：<https://github.com/aliyun/aliyun-oss-python-sdk/tree/master/oss2>

