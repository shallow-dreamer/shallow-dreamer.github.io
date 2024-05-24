---
layout:     post
title:      "poetry创建django项目"
subtitle:   ""
date:       2023-10-30
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - poetry
    - django
---

> 假定当前存在一个目录为{HOME}/ibn-quality,下列操作在当前目录下操作。

- 安装poetry - [打开A黑黑黑的博客](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2FAmio_%2Farticle%2Fdetails%2F126318486)

- 更改poetry配置：`poetry config virtualenvs.in-project true`

- 创建项目目录：`mkdir ibn-quality-be && cd ibn-quality-be`

- 初始化poetry：`poetry init`，可一路回车

- 修改项目基础配置 - `pyproject.toml`

  ```ini
  ini复制代码[tool.poetry]
  name = "ibn-quality-be"
  version = "0.1.0"
  description = "xx区质量委质量后台服务"
  authors = ["xxx <xxx@xiaomi.com>"]
  
  [tool.poetry.dependencies]
  python = "^3.10"
  
  [tool.poetry.dev-dependencies]
  
  [build-system]
  requires = ["poetry-core>=1.0.0"]
  build-backend = "poetry.core.masonry.api"
  
  [[tool.poetry.source]]
  name = 'aliyun.mirrors'
  url = "https://mirrors.aliyun.com/pypi/simple/"
  ```

- [**添加之前最好先换下载源**]添加依赖，如`django`/`djangorestframework` - `poetry add django djangorestframwork`

  ```python
  python复制代码# django项目中的settings.py文件配置
  INSTALLED_APPS = [
      ...
      'rest_framework',
  ]
  ```

- 激活虚拟环境 - `poetry shell`

- 新建`django`项目 - `django-admin startproject ibn_quality_be .`(注意最后有一个点，表示在当前目录下创建)

- 创建`apps`目录 - 应用程序统一目录管理 - `cd ibn-quality-be && mkdir apps`

  ```python
  python复制代码# 需要在{project_path}/ibn-quality-be/settings.py中修改以下部分
  import sys
  from pathlib import Path
  
  # Build paths inside the project like this: BASE_DIR / 'subdir'.
  BASE_DIR = Path(__file__).resolve().parent.parent
  APPS_DIR = Path(BASE_DIR, 'apps')
  sys.path.insert(0, str(APPS_DIR))
  ...
  ```

- 创建应用程序 - `cd apps && django-admin startapp message_template`

```bash
bash复制代码  # 1. 修改应用程序目录下的apps.py
  class MessageTemplateConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.message_template'  # 前面加上`apps.`
    verbose_name = '飞书消息模板'  # 添加该配置，在admin站会使用到
  
  # 2. 添加应用程序到`INSTALLED_APPS`
  INSTALLED_APPS = [
    ...
    'apps.message_template',
  ]
  
  # 最终项目目录如下：/Users/xxx/Work/projects/xiaomi/ibn-quality/ibn-quality-be
  ❯ tree ibn-quality-be -I .venv
  ibn-quality-be
  ├── apps
  │   └── message_template
  │       ├── __init__.py
  │       ├── admin.py
  │       ├── apps.py
  │       ├── migrations
  │       │   └── __init__.py
  │       ├── models.py
  │       ├── tests.py
  │       └── views.py
  ├── ibn_quality_be
  │   ├── __init__.py
  │   ├── asgi.py
  │   ├── settings.py
  │   ├── urls.py
  │   └── wsgi.py
  ├── manage.py
  ├── poetry.lock
  └── pyproject.toml
  
  4 directories, 15 files
```

## AUTH_USER_MODEL - 自定义用户模型

> **注意：必须在新项目开始时就定义，在完成第一次迁移后就不能再次定义用户模型**
>
> cd apps && django-admin startapp user

```python
python复制代码# 以下为自定义用户模型全流程

# 1. settings.py 添加以下配置
INSTALLED_APPS = [
    ...
    'apps.user',
]
# 自定义用户模型
AUTH_USER_MODEL = 'user.UserProfile'

# 2. apps/user/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class UserProfile(AbstractUser):
  	# 该字段仅是个例子,如暂时不需要拓展任意字段，可直接使用`pass`关键字
    avatar = models.CharField(verbose_name='头像', null=True, blank=True, max_length=255)

    class Meta:
        db_table = 'user_info'
        verbose_name = '用户信息表'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.username
      
# 3. apps/user/apps.py
from django.apps import AppConfig

# 项目配置一般在settings.py中配置,各app的配置则在对应app下的apps.py中配置
class UsersConfig(AppConfig):
    # 定义默认主键的数据类型
    default_auto_field = 'django.db.models.BigAutoField'
    # 定义app的完整路径,一般与settings.py中注册的INSTALLED_APPS信息相同
    name = 'apps.user'
    # 别名,在admin站会用到
    verbose_name = '用户管理'
    
# 4. apps/user/admin.py
from django.contrib import admin

from apps.user.models import UserProfile

class UserProfileAdmin(admin.ModelAdmin):
    # admin站需要展示的字段
    list_display = ('username', 'email', 'avatar', 'is_staff', 'is_active', 'date_joined')
    # admin站可点击进入编辑态的字段
    list_display_links = ('username', 'email')

# 注册模型,为了能在admin站能看到该模型
admin.site.register(UserProfile, UserProfileAdmin)
```

## 创建`apps`目录统一管理应用程序

```bash
bash复制代码# 创建apps目录
cd {DJANGO_PROJECT_PATH} && mkdir apps
# 修改settings.py文件 - {DJANGO_PROJECT_PATH}/{PROJECT_NAME}/settings.py
import sys
from pathlib import Path

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent
APPS_DIR = Path(BASE_DIR, 'apps')
sys.path.insert(0, str(APPS_DIR))
......
```

## 限制Host访问 - ALLOWED_HOSTS

```python
python复制代码# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = True

# 作用: 限制请求的Host，防止黑客攻击
# *: 所有的网址都能访问该Django项目
# ['www.example1.com', 'www.example2.com']: 仅配置的Host能访问
# ['*.example.com']: 任意以example.com后缀的Host能访问
# 当DEBUG = True && ALLOWED_HOSTS = []为空时, ['.localhost', '127.0.0.1', '[::1]']能访问
ALLOWED_HOSTS = []
```

## MySQL or MariaDB数据库

### 最低版本要求

```
复制代码mysql：5.7及以上版本
MariaDB：10.3及以上版本
```

### 全局配置字符集

```python
python复制代码# 'OPTIONS': {'charset': 'utf8mb4'}
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'ibn_quality',
        'USER': 'root',
        'PASSWORD': 'IBN_Quality818',
        'HOST': '10.38.160.173',
        'PORT': '3306',
        'OPTIONS': {'charset': 'utf8mb4'},
    }
}
```

### 创建数据库字符集要指定'utf-8'

```mysql
mysql

复制代码create database ibn_quality_test default character set utf8mb4 collate utf8mb4_unicode_ci;
```

### 不允许远程访问 - Host xxx is not allowed to connect to this MySQL server

```mysql
mysql复制代码mysql -u root -p
use mysql;
SELECT `Host`,`User` FROM user;
# 更新权限
update user set host = '%' where user = 'root';
# 强制刷新权限
flush privileges;
```

### django与mysql配置

```python
python复制代码# 1. 安装pymysql
poetry add pymysql
# 2. 在项目应用下的__init__.py配置（与settings.py同级目录）
import pymysql

pymysql.install_as_MySQLdb()
# 3. settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'ibn_quality',
        'USER': 'root',
        'PASSWORD': 'IBN_Quality818',
        'HOST': '10.38.160.173',
        'PORT': '3306',
        'OPTIONS': {'charset': 'utf8mb4'},
    }
}
```

## 时区、语言翻译等问题

```python
python复制代码# 在settings.py中修改
# 修改语言设置为中文
LANGUAGE_CODE = 'zh-hans'

# 解决时间慢8h的问题-修改时区
TIME_ZONE = 'Asia/Shanghai'

USE_I18N = True

# 解决时间慢8h的问题
USE_TZ = False
```

## 解决接口请求携带的参数过多时请求失败的问题

```python
python复制代码# 在settings.py中添加
DATA_UPLOAD_MAX_NUMBER_FIELDS = 1024000
```

## DRF序列化与反序列化

### 名词解释 && 简单例子

> **序列化**：将复杂的数据类型，如`querysets`/`model`实例转化为`python`基础数据类型，方便后续更方便的转化为`JSON`/`XML`或其他类型的上下文
>
> **反序列化**：与序列化相反，是将基础数据类型转化为复杂的数据类型，方便后续校验

- 序列化的🌰：

```python
python复制代码# User是一个`model`类; UserSerializer是一个`Serializer`类
user = User()
# 序列化`model`实例
serializer = UserSerializer(user)
# 调用`.data`得到`python`基础数据类型
print(serializer.data)  # {'name': 'xxx', 'age': xxx}

# 使用`JsonRenderer`类转化为`JSON`
from rest_framework.renderers import JSONRenderer

print(JSONRenderer().render(serializer.data))  # b'{"name": "xxx", "age": xxx}'
```

- 反序列化的🌰：

```python
python复制代码# 还是上面的例子，最后一步改为：
user_json = JSONRenderer().render(serializer.data)
# 官方使用`stream`方式转化，我认为可使用`json`库的`.loads()`一样可转化为`dict`类型
# 以下操作是将json字符串转化为python原始数据类型
import io
from rest_framework.parsers import JSONParser

stream = io.BytesIO(json)
data = JSONParser().parse(stream)

# 将原始数据类型转成serializer实例
serializer = UserSerializer(data=data)
serializer.is_valid()  # 校验是否合法
serializer.validated_data  # {'name': 'xxx', 'age': xxx}
```

### 【推荐】使用`ModelSerializer`序列化

> 相比常规的`Serializer`类有以下3点优点：
>
> 1. 可以基于`model`自动生成`field`的集合，可通过`repr(XXXSerializer)`查看
> 2. 可以自动生成序列化校验，如`unique_together`
> 3. 实现了`.create()`和`.update()`方法

```python
python复制代码class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['name', 'age', 'created']
        depth = 1
# 所有字段均需序列化时使用: fields = '__all__'
# 排除某个字段时使用： exclude = ['age']
# 序列化的深度用`Meta.depth`值来指定
```

## pycharm无法带出`models.objects`问题的解决方案

### 方法一：修改某个app下的models.py文件

```python
python复制代码class QuestionBank(models.Model):
    ...
    objects = models.Manager()
    
    class Meta:
      ...
```

### 方法二：修改`pycharm`配置（社区版好像没有该配置）

`Commond`+ `,`打开`Preferences`，搜索Django，进入`Django`配置，勾选启用django支持

## 接入`Celery`/`django_celery_results`/`django_celery_beat`

> celery: [简单、灵活、可靠的分布式定时任务库](https://link.juejin.cn?target=https%3A%2F%2Fdocs.celeryq.dev%2Fen%2Fmaster%2Findex.html)
>
> django_celery_results: [存储celery执行结果的库](https://link.juejin.cn?target=https%3A%2F%2Fdjango-celery-results.readthedocs.io%2Fen%2Flatest%2F)
>
> Django_celery_beat: [周期性定时任务库](https://link.juejin.cn?target=https%3A%2F%2Fdjango-celery-beat.readthedocs.io%2Fen%2Flatest%2F)
>
> 安装依赖
>
> poetry add celery django_celery_results django_celery_beat

```python
python复制代码# 编辑settings.py
INSTALLED_APPS = [
		...
    'django_celery_results',  # 使用django作为任务执行结果存储
    'django_celery_beat',  # 使admin站支持周期性定时任务展示
]
# Celery相关配置
# 设定时区(在celery.py中通过程序配置,在这边配置不生效,暂未找到原因)
# CELERY_TIMEZONE = TIME_ZONE
# 取消生效UTC
# CELERY_ENABLE_UTC = False
# 解决`MySQL backend does not support timezone-aware datetimes when USE_TZ is False`的问题
DJANGO_CELERY_BEAT_TZ_AWARE = False
# celery-states: pending->(started)->success/failure(retry、revoked)
# started状态仅在CELERY_TASK_TRACK_STARTED设定为true时会被记录
# 默认False.一般无需配置,如果有长时间运行的任务且需要报告任务状态时,可设置为True
CELERY_TASK_TRACK_STARTED = True
# hard模式.如果在指定时间内task未执行结束则强制终止(单位: s)
CELERY_TASK_TIME_LIMIT = 60 * 60
# soft模式.如果在指定时间内task未执行结束则可在task中捕获异常做后续处理(单位: s)
# CELERYD_SOFT_TIME_LIMIT = 60 * 60
# 消息中间件,接受producer发来的消息,并将任务加入队列
CELERY_BROKER_URL = 'redis://:mimarket..@10.38.160.165:6379/1'
# celery任务执行结果保存至django数据库中
CELERY_RESULT_BACKEND = 'django-db'
# 使用django-celery-beat插件需配置.作用:启动beat时命令行可使用-S django即可
CELERYBEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
## --------------------------


# 编辑{project_dir}/{project}/__init__.py
from .celery import app as celery_app

# 保证当django启动时celery app被加载,方便后期@shared_task语法糖可使用
__all__ = ('celery_app',)
## --------------------------


# 新建{project_dir}/{project}/celery.py
# 将django的默认配置引入Celery程序, 必须在创建celery实例之前设定
import os

from celery import Celery
from django.apps import apps

from ibn_quality_be.settings import TIME_ZONE, CELERY_BROKER_URL, CELERY_RESULT_BACKEND

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'ibn_quality_be.settings')

# 创建Celery实例
app = Celery('feedback_be', broker=CELERY_BROKER_URL, backend=CELERY_RESULT_BACKEND)

# 指定django的settings.py配置作为celery的配置源,可以减少多个配置文件
app.config_from_object('django.conf:Settings', namespace='CELERY')

# 自动从已注册的app中下的tasks.py任务
app.autodiscover_tasks(lambda: [n.name for n in apps.get_app_configs()])

app.conf.timezone = TIME_ZONE
app.conf.enable_utc = False
## --------------------------


# 重新migrate
python manage.py migrate django_celery_beat
python manage.py migrate django_celery_results


# 检查新增的表
djang_celery_beat 新增6张彪
django_celery_results 新增3张表
```



作者：A黑黑黑
链接：https://juejin.cn/post/7208071684590714917
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。





好的，以下是一个完整的示例代码，包括 Django 项目的配置和应用代码：

1. **创建 Django 项目并配置 Poetry**：

首先，确保你已经安装了 Poetry，并在命令行中执行以下步骤：

```bash
# 创建一个新的 Django 项目目录
mkdir my_django_project
cd my_django_project

# 使用 Poetry 创建虚拟环境
poetry init -n

# 安装 Django
poetry add django

# 创建 Django 项目
poetry run django-admin startproject my_project .
```

2. **安装并配置 Django REST framework 和 PostgreSQL**：

继续在命令行中执行以下步骤：

```bash
# 安装 Django REST framework 和 PostgreSQL 连接库
poetry add djangorestframework psycopg2-binary

# 在 settings.py 中配置 'rest_framework' 和数据库连接
```

在 `my_project/settings.py` 文件中进行配置：

```python
# my_project/settings.py

# 添加 'rest_framework' 到 INSTALLED_APPS
INSTALLED_APPS = [
    ...
    'rest_framework',
]

# 配置 PostgreSQL 数据库连接
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'my_database',  # 数据库名称
        'USER': 'my_user',      # 数据库用户名
        'PASSWORD': 'my_password',  # 数据库密码
        'HOST': 'localhost',    # 数据库主机地址
        'PORT': '5432',         # 数据库端口
    }
}
```

3. **创建 Django 应用**：

继续在命令行中执行以下步骤：

```bash
# 创建一个新的 Django 应用
poetry run python manage.py startapp my_app
```

4. **编写 Django 应用中的视图和模型**：

在 `my_app` 目录下的 `views.py` 中编写视图和 `models.py` 中编写模型。

```python
# my_app/views.py
from rest_framework import generics
from .models import MyModel
from .serializers import MyModelSerializer

class MyModelListView(generics.ListCreateAPIView):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer

class MyModelDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer
```

```python
# my_app/models.py
from django.db import models

class MyModel(models.Model):
    name = models.CharField(max_length=100)
    description = models.TextField()

    def __str__(self):
        return self.name
```

5. **编写序列化器**：

在 `my_app` 目录下创建 `serializers.py` 文件，并编写序列化器。

```python
# my_app/serializers.py
from rest_framework import serializers
from .models import MyModel

class MyModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = MyModel
        fields = '__all__'
```

6. **配置 URL 路由**：

在 `my_project` 目录下的 `urls.py` 中配置 URL 路由。

```python
# my_project/urls.py
from django.urls import path, include

urlpatterns = [
    path('api/', include('my_app.urls')),
]
```

在 `my_app` 目录下创建 `urls.py` 文件，并配置应用的 URL 路由。

```python
# my_app/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('mymodel/', views.MyModelListView.as_view(), name='mymodel-list'),
    path('mymodel/<int:pk>/', views.MyModelDetailView.as_view(), name='mymodel-detail'),
]
```

7. **迁移数据库**：

```bash
# 创建数据库表
poetry run python manage.py migrate
```

8. **运行开发服务器**：

```bash
poetry run python manage.py runserver
```

现在你已经拥有一个使用 Poetry 管理依赖的 Django 后端项目，其中包含了 Django REST framework 和 PostgreSQL 支持，可以根据需要继续添加自己的应用和功能。

如果你可以直接访问接口并获取信息，但访问 API 文档时遇到 403 错误，问题可能出在 API 文档的权限配置或相关设置上。下面是一些排查和解决这个问题的方法：

### 1. 确认 API 文档 URL 配置

首先，确保你的 `urls.py` 文件中正确配置了 API 文档的 URL。下面是一个例子，使用 `include_docs_urls` 来生成 API 文档：

```python
# my_project/urls.py

from django.contrib import admin
from django.urls import path, include
from rest_framework.documentation import include_docs_urls

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('my_app.urls')),
    path('api/docs/', include_docs_urls(title='API 文档')),
]
```

### 2. 检查 REST framework 的设置

确保在 `settings.py` 中的 `REST_FRAMEWORK` 配置中没有设置过于严格的权限限制。默认情况下，API 文档应该是可以被所有用户访问的：

```python
# my_project/settings.py

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny',
    ],
}
```

### 3. 确认 CSRF 设置

如果你使用了 Session 认证或其他可能涉及 CSRF 保护的认证方式，确保你的 CSRF 设置不会阻止访问 API 文档。你可以尝试暂时禁用 CSRF 检查以进行排查：

```python
# my_project/settings.py

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    # 'django.middleware.csrf.CsrfViewMiddleware',  # 暂时注释掉
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',  # 确保它在正确的位置
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

### 4. 检查 Django 和 REST framework 版本兼容性

确保你使用的 Django 和 Django REST framework 版本是兼容的。有时版本不兼容可能会导致一些意外问题。你可以通过 `pip show django djangorestframework` 查看安装的版本，并参阅各自的文档确认兼容性。

### 5. 尝试使用 Swagger 或 Redoc

如果 `include_docs_urls` 仍然无法正常工作，你可以尝试使用 `drf-yasg` 生成 Swagger 文档或 `drf-spectacular` 生成 OpenAPI 文档：

**安装 drf-yasg:**

```bash
poetry add drf-yasg
```

**配置 drf-yasg:**

```python
# my_project/urls.py

from django.contrib import admin
from django.urls import path, include
from rest_framework import permissions
from drf_yasg.views import get_schema_view
from drf_yasg import openapi

schema_view = get_schema_view(
    openapi.Info(
        title="API 文档",
        default_version='v1',
    ),
    public=True,
    permission_classes=(permissions.AllowAny,),
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('my_app.urls')),
    path('swagger/', schema_view.with_ui('swagger', cache_timeout=0), name='schema-swagger-ui'),
    path('redoc/', schema_view.with_ui('redoc', cache_timeout=0), name='schema-redoc-ui'),
]
```

### 6. 检查服务器和浏览器设置

如果你在本地开发环境中没有问题，但在生产环境中遇到问题，检查你的服务器和浏览器设置，例如：

- 确认服务器没有设置限制访问特定路径或 IP 的规则。
- 确认浏览器没有缓存旧的 CSRF token 或其他认证信息，尝试清除浏览器缓存或使用隐身模式访问。

通过上述步骤，你应该能够找出并解决访问 API 文档时遇到的 403 错误问题。如果问题仍然存在，请提供更多的错误日志或详细信息，以便进一步排查。

如果你使用了 `drf-yasg` 来生成 Swagger 文档，并且在访问时遇到了 403 错误，可以考虑以下几种可能的解决方法：

1. **确认权限设置**：
   确保你在 `settings.py` 中配置了适当的权限设置，以确保所有用户都能够访问 Swagger 文档。在 `drf-yasg` 中，默认情况下，Swagger 文档是可以被所有用户访问的。

```python
# my_project/settings.py

INSTALLED_APPS = [
    ...
    'drf_yasg',
]

# Swagger settings
SWAGGER_SETTINGS = {
    'DEFAULT_INFO': 'my_project.urls.swagger_info',
    'SECURITY_DEFINITIONS': {
        'Bearer': {
            'type': 'apiKey',
            'name': 'Authorization',
            'in': 'header',
        }
    },
}
```

2. **检查 URL 配置**：
   确保你的 `urls.py` 文件中正确配置了 Swagger 文档的 URL 地址，并且没有设置任何权限限制。

```python
# my_project/urls.py

from django.contrib import admin
from django.urls import path, include
from rest_framework import permissions
from drf_yasg.views import get_schema_view

schema_view = get_schema_view(
    openapi.Info(
        title="API 文档",
        default_version='v1',
    ),
    public=True,
    permission_classes=(permissions.AllowAny,),  # 确保这里使用了 AllowAny 权限
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('my_app.urls')),
    path('swagger/', schema_view.with_ui('swagger', cache_timeout=0), name='schema-swagger-ui'),
]
```

3. **排查认证和权限设置**：
   如果你的 API 需要认证才能访问，确保你的请求包含了正确的认证信息，例如 Token 或用户名密码。你可以尝试使用无认证的方式访问 Swagger 文档，看是否仍然遇到 403 错误。

4. **检查其它中间件和安全设置**：
   确保你的项目中没有设置了过于严格的中间件或安全设置，以致于阻止了对 Swagger 文档的访问。你可以尝试暂时禁用一些中间件或安全设置来排查问题。

通过以上步骤，你应该能够解决访问 Swagger 文档时遇到的 403 错误问题。如果问题仍然存在，请提供更多的详细信息以便进一步排查。

要设置 API 的访问权限，并将其连接到单点登录（SSO）系统，可以使用 Django REST Framework（DRF）提供的权限类和认证类，并将其与 SSO 提供的认证机制集成。

以下是一般的步骤：

### 1. 配置认证类和权限类

首先，在 Django 的 `settings.py` 文件中配置 DRF 的认证类和权限类。

```python
# settings.py

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
        'yourapp.authentication.SSOAuthentication',  # 自定义的SSO认证类
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',  # 仅允许已认证用户访问
    ],
}
```

在这里，你需要创建一个自定义的 SSO 认证类，以实现与你的单点登录系统的集成。你可以继承 DRF 的 `BaseAuthentication` 类并实现 `authenticate` 方法来实现自定义认证逻辑。

### 2. 实现单点登录认证逻辑

在你的应用程序中创建一个自定义的认证类，并实现单点登录的认证逻辑。

```python
# yourapp/authentication.py

from rest_framework.authentication import BaseAuthentication
from rest_framework.exceptions import AuthenticationFailed

class SSOAuthentication(BaseAuthentication):
    def authenticate(self, request):
        # 在这里实现单点登录的认证逻辑
        # 检查请求中是否包含了单点登录的身份认证信息
        # 如果认证成功，返回用户和认证信息元组 (user, auth)，
        # 如果认证失败，抛出 AuthenticationFailed 异常
        # 示例代码：
        # if sso_login_successful(request):
        #     return (user, auth)
        # else:
        #     raise AuthenticationFailed('SSO authentication failed.')
        pass
```

### 3. 集成单点登录系统

根据你所使用的单点登录系统，使用相应的库或工具来集成单点登录系统，并实现 `SSOAuthentication` 类中的认证逻辑。

通常，你需要获取从单点登录系统返回的身份认证信息（例如 token），并使用该信息来验证用户身份。你可能需要与单点登录系统的 API 进行通信来验证 token 的有效性，并获取与用户相关的信息。

### 4. 测试认证和权限设置

确保在实现认证和权限设置后进行测试，以确保 API 正确地与单点登录系统集成，并且只允许经过认证的用户访问受保护的资源。

你可以使用 Postman 或类似的工具来发送请求，并验证 API 的行为是否符合预期。

通过以上步骤，你应该能够将 API 链接到单点登录系统，并设置适当的认证和权限，以确保 API 的安全性和可访问性。
