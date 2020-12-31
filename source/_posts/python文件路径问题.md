---
title: python文件路径问题
top: false
cover: false
toc: true
mathjax: true
date: 2020-12-31 09:46:31
password:
summary:
tags:
categories:
---

# python路径问题

## 路径描述和解决方案

之前在小同志编写的项目中，出现了下面的一段代码:

```python
X_train, y_train, X_test, y_test, scaler = load_dataset(r'../data/test.csv')

```
调用的load_dataset方法如下：

```python
def load_dataset(DATASET_PATH):
    seq_length = 100
    split = 0.8
    print(DATASET_PATH)
    print(os.path.exists(DATASET_PATH))
    if os.path.exists(DATASET_PATH):
        df = pd.read_csv(DATASET_PATH)
    else:
        raise FileNotFoundError('File %s not found!' % DATASET_PATH)

    ......

```

在python程序运行的过程中，诡异的事情发生了，具体体现在：明明文件就在相对路径的位置放着，通过shell执行命令可以切换到对应的文件夹，也能正常通过cat命令输出文件信息，但是在python程序中，业务逻辑始终停在自定义异常上。

环境信息如下，程序运行的根目录为*/home/python/app*，该目录下有两个文件夹，一个为src，是代码所在的位置，另一个是data，也就是test.csv文件所在位置。

下面开始尝试排查问题，在调用load_dataset()方法之前,故意输出多个路径信息对比，代码如下：

```python

...

print("1:", os.path.abspath(r'../data/test.csv'))
# 添加这一句是为了更快的输出到docker日志信息中
sys.stdout.flush()

...
```

输出信息如下：
```

1:/home/python/data/test.csv

```

在输出的内容中竟然少了**app**这个文件夹，这个运行结果出现在docker容器中，容器运行的系统是ubuntu。但是对比在windows下，同样的路径是可以访问到文件的。至于为何那么诡异，丢失了路径，目前暂时无法解释。

下一步进行了更多的测试，如下：

```python

...

basedir = os.path.abspath(os.path.dirname(__file__))
print("2:", basedir)
sys.stdout.flush()
print("1:", os.path.abspath(r'../data/test.csv'))
sys.stdout.flush()
print("3：", basedir + r'../data/test.csv')
sys.stdout.flush()

...

```

输出信息如下：
```

2: /home/python/app/src
1: /home/python/data/test.csv
3： /home/python/app/src../data/test.csv


```

但是还是发现不能解决这个问题，尝试进一步测试，如下：

```python

...

basedir = os.path.abspath(os.path.dirname(__file__))
print("2:", basedir)
sys.stdout.flush()
print("1:", os.path.abspath(r'../data/test.csv'))
sys.stdout.flush()
print("3：", basedir + r'../data/test.csv')
sys.stdout.flush()
print("4: ", os.path.abspath(basedir + r'/'+ r'../data/test.csv'))

...

```

输出信息如下：
```

2: /home/python/app/src
1: /home/python/data/test.csv
3： /home/python/app/src../data/test.csv
4:  /home/python/app/src/data/test.csv

```

在使用第四种方式的时候，正常可以定位到文档所在位置。

于是最终解决方式如下：

```python

X_train, y_train, X_test, y_test, scaler = load_dataset(os.path.abspath(basedir + r'/'+ r'../data/test.csv'))

```

## 参考地址

* https://blog.csdn.net/qq_36711420/article/details/79631141
* https://blog.csdn.net/qq_34814495/article/details/109682792