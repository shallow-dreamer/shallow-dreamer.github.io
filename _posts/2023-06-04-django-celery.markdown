---
layout:     post
title:      "django-celery"
subtitle:   ""
date:       2023-06-04
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - django
    - celery
---

`celery_app.config_from_object('django.conf.settings')` 是用来将 Django 项目的配置加载到 Celery 应用程序中的。如果执行该语句时出现问题，可能有以下几个原因：

1. Django 项目没有正确配置

   确保您的 Django 项目已经正确配置。在 settings.py 文件中，必须包含一些与 Celery 相关的配置项，例如 `CELERY_BROKER_URL` 和 `CELERY_RESULT_BACKEND` 等。

2. 无法找到 Django 项目

   如果执行 `celery_app.config_from_object('django.conf.settings')` 时出现错误，可能是因为无法找到 Django 项目。请确保您在正确的位置运行 Celery worker，以便能够找到 Django 项目并加载其配置。

3. 问题出现在其他地方

   如果上述两个步骤都没有解决问题，那么问题可能出现在其他地方。建议检查 Celery 的日志文件以获取更多信息，或者提供更多的错误信息以便更好地帮助您。

下面是一个示例配置，可以参考一下：

```python
# 在 settings.py 文件中添加如下配置

CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'

CELERY_TASK_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_RESULT_SERIALIZER = 'json'

CELERY_IMPORTS = (
    'myapp.tasks',
)
```

在启动 Celery worker 时，需要指定 Django 项目的配置模块，并将其传递给 `config_from_object` 方法，例如：

```bash
celery -A myproject worker --loglevel=info --settings=myproject.settings
```

其中，`myproject` 是您的 Django 项目名称，`settings` 是 Django 项目中的配置模块。在代码中，可以使用如下语句加载配置：

```python
app = Celery('myapp')
app.config_from_object('django.conf:settings', namespace='CELERY')
```

以下是一个使用 Django 和 Celery 的项目 demo，可以在 Python 3.11 版本中运行：

1. 创建 Django 项目

   首先创建一个 Django 项目，可以使用如下命令：

   ```bash
   django-admin startproject myproject
   ```

2. 安装 Celery

   在项目目录下使用 pip 工具安装 Celery：

   ```bash
   pip install celery
   ```

3. 配置 Celery

   在 Django 项目的 settings.py 中添加 Celery 相关配置：

   ```python
   # 指定 broker
   CELERY_BROKER_URL = 'redis://localhost:6379/0'

   # 设置结果存储
   CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'

   # 指定任务模块路径
   CELERY_IMPORTS = ('myapp.tasks',)
   ```

4. 创建任务

   在 myapp 应用下创建 tasks.py 文件，定义一个简单的任务：

   ```python
   from celery import shared_task
   
   @shared_task
   def add(x, y):
       return x + y
   ```

5. 启动 worker

   在项目根目录下创建一个 celery.py 文件，启动 worker：

   ```python
   from myproject import settings
   from celery import Celery
   
   app = Celery('myproject')
   
   # 加载配置
   app.conf.update(settings.__dict__.get('CELERY_CONFIG', {}))
   
   if __name__ == '__main__':
       app.start()
   ```

   启动 worker：

   ```bash
   celery -A myproject worker -l info
   ```

6. 调用任务

   在视图函数中调用任务：

   ```python
   from django.http import JsonResponse
   from myapp.tasks import add
   
   def myview(request):
       result = add.delay(1, 2)
       return JsonResponse({'task_id': result.id})
   ```

7. 查看任务结果

   可以通过异步方式获取任务结果：

   ```python
   from myapp.tasks import add
   
   result = add.delay(1, 2)
   print(result.id)  # 查看任务 id
   print(result.get())  # 获取任务结果（会阻塞当前线程）
   ```

   或者使用回调函数获取任务结果：

   ```python
   from myapp.tasks import add
   
   def on_task_complete(result):
       print(result)
   
   result = add.delay(1, 2)
   result.then(on_task_complete)
   ```

这个 demo 展示了如何在 Django 中使用 Celery 来执行异步任务，可以根据自己的需求进行进一步扩展。需要注意的是，实际应用中还需要对 Celery 进行更细致的配置，例如设置 worker 数量、超时时间等。
