---
layout:     post
title:      "celery的self问题"
subtitle:   "celery传参时参数为self时的问题"
date:       2023-04-21
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - python
    - celery
---

不用filter=task_method时，实例(self)不会自动传入

bind=True时， task对象会作为第一个参数自动传入

加上filter=task_method参数，实例(self)会作为第一个参数自动传入

加上filter=task_method, bind=True, task对象会作为第一个，实例(self)会作为第二个参数自动传入。

所以，最佳调用方式应为:       

```python
from celery.contrib.methods import task_method 

class A(object):           
    def __init__(self):  
        object.__init__(self) 
        self.a = 1         
        self.b = 2  
        
    @app.task(bind=True, filter=task_method)  
    def test1(task_self, self, a, b):                   
        print a
        print b          
        return a+b+self.a+self.b
```

或：                                             

```python
from celery.contrib.methods import task_method

class A(object):
    def __init__(self): 
        object.__init__(self) 
        self.a = 1      
        self.b = 2    

    @app.task(filter=task_method) 
    def test1(self, a, b):
        print a          
        print b          
        return a+b+self.a+self.b
```
