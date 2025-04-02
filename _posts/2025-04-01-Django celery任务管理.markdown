---
layout:     post
title:      "Django celery任务管理"
subtitle:   ""
date:       2025-04-01
author:     " Shallow Dreamer"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Django
    - celery
---

好的，我会提供一个**完整的 Django + Celery 任务调度系统**，它支持以下功能：

✅ **一次性任务**（指定时间执行一次）
 ✅ **周期任务**（每隔固定时间执行一次）
 ✅ **定时任务**（每天某个时间点执行）
 ✅ **任务启用/禁用**
 ✅ **支持 Django Admin 和 REST API 进行任务管理**
 ✅ **可扩展，支持不同类型的 Celery 任务**

------

# **📌 目录结构**

```bash
your_project/
│── your_project/            # Django 项目
│   │── settings.py          # Django 配置
│   │── urls.py              # URL 路由
│   └── celery.py            # Celery 配置
│
│── your_app/                # Django 应用
│   │── models.py            # 任务数据库模型
│   │── tasks.py             # Celery 任务
│   │── views.py             # 任务 API 视图
│   │── serializers.py       # Django REST 框架序列化
│   │── admin.py             # Django Admin 配置
│   └── urls.py              # 任务 API 路由
│
│── manage.py                # Django 入口文件
│── requirements.txt         # 依赖列表
│── celery_worker.sh         # Celery 启动脚本
└── README.md                # 项目说明
```

------

# **1️⃣ 安装 Celery 和 Redis**

先安装 Celery、Redis 和 Django REST Framework：

```bash
pip install celery redis django djangorestframework
```

Redis 需要单独安装，运行：

```bash
sudo apt install redis  # Ubuntu
brew install redis  # macOS
```

------

# **2️⃣ 配置 Celery**

在 `your_project/celery.py` 添加 Celery 配置：

```python
import os
from celery import Celery

# 设置 Django 的 settings 模块
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'your_project.settings')

app = Celery('your_project')

# 从 Django 配置文件中读取 Celery 配置
app.config_from_object('django.conf:settings', namespace='CELERY')

# 自动发现所有 Django app 中的 tasks.py
app.autodiscover_tasks()
```

在 `your_project/settings.py` 里添加 Celery 配置：

```python
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
```

------

# **3️⃣ 定义任务模型**

在 `your_app/models.py` 定义任务存储：

```python
from django.db import models

class ScheduledTask(models.Model):
    TASK_TYPES = (
        ('one_time', '一次性任务'),
        ('interval', '间隔任务'),
        ('cron', '定时任务'),
    )

    name = models.CharField(max_length=255, unique=True, verbose_name="任务名称")
    task_name = models.CharField(max_length=255, verbose_name="Celery 任务路径")
    task_type = models.CharField(max_length=10, choices=TASK_TYPES, verbose_name="任务类型")
    run_at = models.DateTimeField(null=True, blank=True, verbose_name="执行时间")  # 仅适用于一次性任务
    interval_seconds = models.IntegerField(null=True, blank=True, verbose_name="间隔秒数")  # 仅适用于间隔任务
    cron_schedule = models.CharField(max_length=50, null=True, blank=True, verbose_name="Cron 表达式")  # 仅适用于定时任务
    args = models.JSONField(default=list, verbose_name="任务参数")
    enabled = models.BooleanField(default=True, verbose_name="是否启用")

    def __str__(self):
        return self.name
```

------

# **4️⃣ 创建 Celery 任务**

在 `your_app/tasks.py` 里定义任务：

```python
from celery import shared_task

@shared_task
def example_task(message):
    print(f"执行任务: {message}")
    return f"任务完成: {message}"

@shared_task
def check_and_run_tasks():
    """定期检查数据库中的任务，并触发 Celery 执行"""
    from your_app.models import ScheduledTask
    from django.utils.timezone import now
    from datetime import datetime
    import crontab

    tasks = ScheduledTask.objects.filter(enabled=True)
    
    for task in tasks:
        if task.task_type == "one_time" and task.run_at and task.run_at <= now():
            example_task.apply_async(args=task.args)
            task.enabled = False  # 只执行一次
            task.save()

        elif task.task_type == "interval" and task.interval_seconds:
            example_task.apply_async(args=task.args, countdown=task.interval_seconds)

        elif task.task_type == "cron" and task.cron_schedule:
            cron = crontab.CronTab(task.cron_schedule)
            if cron.next(default_utc=True) == 0:
                example_task.apply_async(args=task.args)
```

**解释：**

- `one_time` 任务：只执行一次，完成后自动禁用。
- `interval` 任务：每 `interval_seconds` 运行一次。
- `cron` 任务：根据 `cron_schedule` 执行。

------

# **5️⃣ 配置 Celery Worker**

在 `your_project/celery.py` 里添加：

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    "check_scheduled_tasks": {
        "task": "your_app.tasks.check_and_run_tasks",
        "schedule": crontab(minute="*"),  # 每分钟执行
    },
}
```

运行 Celery：

```bash
celery -A your_project worker --loglevel=info
```

------

# **6️⃣ Django Admin 管理任务**

在 `your_app/admin.py` 里注册：

```python
from django.contrib import admin
from .models import ScheduledTask

@admin.register(ScheduledTask)
class ScheduledTaskAdmin(admin.ModelAdmin):
    list_display = ("name", "task_type", "enabled", "updated_at")
    search_fields = ("name",)
```

------

# **7️⃣ 创建 Django API**

在 `your_app/views.py` 里：

```python
from rest_framework import viewsets
from .models import ScheduledTask
from .serializers import ScheduledTaskSerializer
from rest_framework.response import Response
from rest_framework.decorators import action

class ScheduledTaskViewSet(viewsets.ModelViewSet):
    queryset = ScheduledTask.objects.all()
    serializer_class = ScheduledTaskSerializer

    @action(detail=True, methods=["post"])
    def run_task(self, request, pk=None):
        """手动触发任务"""
        task = self.get_object()
        from your_app.tasks import example_task
        example_task.apply_async(args=task.args)
        return Response({"status": "任务已触发"})
```

在 `your_app/serializers.py` 里：

```python
from rest_framework import serializers
from .models import ScheduledTask

class ScheduledTaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = ScheduledTask
        fields = "__all__"
```

在 `your_project/urls.py` 里：

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from your_app.views import ScheduledTaskViewSet

router = DefaultRouter()
router.register(r"tasks", ScheduledTaskViewSet)

urlpatterns = [
    path("api/", include(router.urls)),
]
```

------

# **8️⃣ 运行项目**

1. 运行 Django：

   ```bash
   python manage.py runserver
   ```

2. 运行 Celery Worker：

   ```bash
   celery -A your_project worker --loglevel=info
   ```

3. 运行 Celery Beat（如果使用 `cron` 任务）：

   ```bash
   celery -A your_project beat --loglevel=info
   ```

------

# **✅ 结果**

- 访问 `/admin/` 可管理任务。
- 访问 `/api/tasks/` 可创建/修改任务。
- Celery **自动执行任务** 🎉

这个方案**可扩展、支持各种任务类型**，并能通过 Django Admin 或 API 进行管理！🚀

是的，你创建的任务会存储在**数据库**中，具体来说，所有任务数据都会存入 `ScheduledTask` 这个数据库表里。

------

# **📌 任务存储方式**

在 `your_app/models.py` 里，我们定义了 `ScheduledTask` 模型，Django 会自动将任务数据存入数据库：

```python
class ScheduledTask(models.Model):
    TASK_TYPES = (
        ('one_time', '一次性任务'),
        ('interval', '间隔任务'),
        ('cron', '定时任务'),
    )

    name = models.CharField(max_length=255, unique=True, verbose_name="任务名称")
    task_name = models.CharField(max_length=255, verbose_name="Celery 任务路径")
    task_type = models.CharField(max_length=10, choices=TASK_TYPES, verbose_name="任务类型")
    run_at = models.DateTimeField(null=True, blank=True, verbose_name="执行时间")  # 仅适用于一次性任务
    interval_seconds = models.IntegerField(null=True, blank=True, verbose_name="间隔秒数")  # 仅适用于间隔任务
    cron_schedule = models.CharField(max_length=50, null=True, blank=True, verbose_name="Cron 表达式")  # 仅适用于定时任务
    args = models.JSONField(default=list, verbose_name="任务参数")
    enabled = models.BooleanField(default=True, verbose_name="是否启用")

    def __str__(self):
        return self.name
```

------

# **📌 任务存在哪？**

任务数据存储在 Django **数据库**（如 PostgreSQL 或 MySQL）中：

1. **如果你使用 SQLite**，任务数据会存入 `db.sqlite3` 文件。
2. **如果你使用 PostgreSQL**，任务数据会存入 `scheduled_task` 表。

你可以通过 **Django Admin** 直接查看这些任务，也可以在数据库里执行 SQL 查询：

```sql
SELECT * FROM your_app_scheduledtask;
```

------

# **📌 任务创建方式**

## **方式 1️⃣：Django Admin 界面**

访问 `/admin/`，可以**手动创建和管理任务**。

------

## **方式 2️⃣：REST API**

可以通过 **REST API** 来创建任务：

```bash
curl -X POST http://127.0.0.1:8000/api/tasks/ \
     -H "Content-Type: application/json" \
     -d '{
         "name": "测试任务",
         "task_name": "your_app.tasks.example_task",
         "task_type": "one_time",
         "run_at": "2025-04-01T12:00:00Z",
         "args": ["Hello, Celery!"],
         "enabled": true
     }'
```

这样，任务会被存入数据库，Celery Worker 也会自动执行它。

------

# **📌 任务如何执行？**

**Celery Worker** 每隔 1 分钟**检查数据库**，然后触发符合条件的任务：

```python
@shared_task
def check_and_run_tasks():
    """定期检查数据库中的任务，并触发 Celery 执行"""
    from your_app.models import ScheduledTask
    from django.utils.timezone import now
    from datetime import datetime
    import crontab

    tasks = ScheduledTask.objects.filter(enabled=True)
    
    for task in tasks:
        if task.task_type == "one_time" and task.run_at and task.run_at <= now():
            example_task.apply_async(args=task.args)
            task.enabled = False  # 只执行一次
            task.save()

        elif task.task_type == "interval" and task.interval_seconds:
            example_task.apply_async(args=task.args, countdown=task.interval_seconds)

        elif task.task_type == "cron" and task.cron_schedule:
            cron = crontab.CronTab(task.cron_schedule)
            if cron.next(default_utc=True) == 0:
                example_task.apply_async(args=task.args)
```

这样，数据库里的任务**会被 Celery 自动执行**，而且任务状态也会更新！

------

# **📌 如何查看已存储的任务？**

你可以在 **Django Admin** 里查看：

1. 访问 `http://127.0.0.1:8000/admin/`
2. 进入 `ScheduledTask` 表
3. 查看所有已存储的任务

或者直接用 **数据库查询**：

```sql
SELECT * FROM your_app_scheduledtask;
```

------

# **✅ 总结**

- **任务会存入数据库 `ScheduledTask` 表**
- **支持 Django Admin 管理任务**
- **支持 REST API 创建任务**
- **Celery Worker 每分钟检查数据库，自动执行任务**

这样，你就可以**在数据库里动态管理 Celery 任务**，不需要重启 Celery Worker！🚀

是的，如果 **Celery Worker 每分钟轮询数据库** 来检查任务，**数据库查询频繁** 可能会影响性能，尤其是当任务量很大时。

## **📌 存在的问题**

1. **高频查询数据库**：每分钟轮询数据库，随着任务量增加，查询负担会变重。
2. **Celery Worker 需要主动检查**：这种轮询方式并不是最优的，可能会浪费计算资源。
3. **任务触发可能会有延迟**：如果任务时间点正好落在两个查询间隔之间，可能会延迟执行。

------

## **✅ 更高效的方式**

我们可以改进：

1. **使用 Celery Beat 来管理周期任务**（官方推荐）
2. **使用数据库触发器或 Webhook 让 Celery Worker 在任务更新时自动触发**
3. **使用 Django Signals 监听任务变化**

------

# **方案 1️⃣：使用 Celery Beat（官方推荐）**

**Celery Beat** 允许你把**定时任务存入数据库**，然后自动触发 Celery 执行，而不需要手动轮询数据库。

### **📌 安装 Celery Beat**

```bash
pip install django-celery-beat
```

### **📌 配置 `INSTALLED_APPS`**

在 `settings.py` 里添加：

```python
INSTALLED_APPS = [
    ...,
    'django_celery_beat',
]
```

### **📌 运行数据库迁移**

```bash
python manage.py migrate django_celery_beat
```

### **📌 在 `your_project/celery.py` 里添加**

```python
from celery.schedules import crontab
from django_celery_beat.models import PeriodicTask, IntervalSchedule

app.conf.beat_scheduler = 'django_celery_beat.schedulers.DatabaseScheduler'
```

### **📌 运行 Celery Beat**

```bash
celery -A your_project beat --loglevel=info
```

### **📌 这样改进了什么？**

✅ **任务存入数据库，Celery Beat 读取数据库并触发任务**
 ✅ **不需要手动轮询数据库，减少 Worker 负担**
 ✅ **任务可以在 Django Admin 里管理（`django_celery_beat` 提供的功能）**

------

# **方案 2️⃣：使用 Django Signals**

**Django Signals 可以在任务创建或更新时，** **自动触发 Celery Worker**，避免不必要的轮询。

在 `your_app/signals.py` 里：

```python
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import ScheduledTask
from .tasks import example_task
from django.utils.timezone import now

@receiver(post_save, sender=ScheduledTask)
def trigger_task_on_create(sender, instance, **kwargs):
    """当任务创建或更新时，自动触发 Celery 执行"""
    if instance.enabled:
        if instance.task_type == "one_time" and instance.run_at and instance.run_at <= now():
            example_task.apply_async(args=instance.args)
            instance.enabled = False  # 只执行一次
            instance.save()
```

### **📌 这样改进了什么？**

✅ **任务创建后自动触发，不需要轮询数据库**
 ✅ **Celery Worker 只执行真正需要的任务**
 ✅ **减少数据库压力，提高效率**

------

# **方案 3️⃣：使用 PostgreSQL 触发器**

如果你的数据库是 **PostgreSQL**，可以使用 **触发器** 让 Celery 在任务更新时自动执行，而不是让 Celery Worker 轮询数据库。

### **📌 1. 创建数据库触发器**

在 PostgreSQL 里执行：

```sql
CREATE FUNCTION notify_task_created() RETURNS TRIGGER AS $$
BEGIN
    NOTIFY new_task, json_build_object('id', NEW.id, 'task_name', NEW.task_name)::text;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER task_trigger
AFTER INSERT OR UPDATE ON your_app_scheduledtask
FOR EACH ROW EXECUTE FUNCTION notify_task_created();
```

### **📌 2. 在 Celery Worker 监听通知**

修改 Celery Worker，让它监听 `new_task` 事件：

```python
import psycopg2
import select
from your_app.tasks import example_task

def listen_for_new_tasks():
    conn = psycopg2.connect("dbname=yourdb user=youruser password=yourpassword")
    conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)
    cur = conn.cursor()
    cur.execute("LISTEN new_task;")

    print("Listening for new tasks...")
    while True:
        select.select([conn], [], [], None)
        conn.poll()
        while conn.notifies:
            notify = conn.notifies.pop(0)
            task_data = json.loads(notify.payload)
            example_task.apply_async(args=[task_data["task_name"]])

listen_for_new_tasks()
```

### **📌 这样改进了什么？**

✅ **任务插入或更新时，数据库主动通知 Celery，不用轮询数据库**
 ✅ **低延迟，任务几乎立即触发**
 ✅ **适合大规模任务管理，减少数据库查询负担**

------

# **📌 总结**

| 方案                  | 适用场景         | 优点                          | 缺点                        |
| --------------------- | ---------------- | ----------------------------- | --------------------------- |
| **Celery Beat**       | 需要周期性任务   | 任务存数据库，自动调度        | 仍需 `beat` 进程            |
| **Django Signals**    | 任务手动创建较多 | 任务创建时自动执行，无需轮询  | 仅适用于任务创建时触发      |
| **PostgreSQL 触发器** | 大规模任务管理   | 数据库主动通知 Celery，低延迟 | 需要 PostgreSQL，配置较复杂 |

------

# **🚀 推荐方案**

- **如果你使用周期性任务（interval/cron）** 👉 用 `Celery Beat`
- **如果你需要任务创建后立刻执行** 👉 用 `Django Signals`
- **如果你有大量任务，数据库性能要最优** 👉 用 `PostgreSQL 触发器`

这样，你的 Celery Worker **不会一直轮询数据库**，避免性能问题，同时还能确保任务能及时执行！ 🚀

下面是一个**完整的 Django + Celery + Celery Beat 解决方案**，它支持 **一次性任务、定时任务（Cron）、间隔任务（Interval）**，并且可以通过 **Django Admin 或 REST API** 进行管理，同时具有**高扩展性**。

------

# **📌 主要技术栈**

- **Celery**：执行异步任务
- **Celery Beat**：管理周期性任务（存入数据库）
- **Redis**：作为 Celery Broker（消息队列）
- **Django REST Framework**（可选）：用于 API 管理任务

------

# **1️⃣ 安装必要的依赖**

```bash
pip install celery redis django-celery-beat django-celery-results djangorestframework
```

------

# **2️⃣ 配置 Django Settings**

在 `settings.py` 里添加 Celery 和 Celery Beat 配置：

```python
from celery.schedules import crontab

# Celery 配置
CELERY_BROKER_URL = 'redis://localhost:6379/0'  # 使用 Redis 作为消息队列
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'

# Celery 结果存储
CELERY_RESULT_BACKEND = 'django-db'

# Celery Beat 配置
INSTALLED_APPS = [
    ...,
    'django_celery_beat',
    'django_celery_results',  # 存储任务结果
]
```

然后运行数据库迁移：

```bash
python manage.py migrate django_celery_beat
python manage.py migrate django_celery_results
```

------

# **3️⃣ 配置 Celery**

创建 `your_project/celery.py` 文件：

```python
import os
from celery import Celery

# 设置 Django 的默认设置模块
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'your_project.settings')

app = Celery('your_project')

# 从 settings.py 加载配置
app.config_from_object('django.conf:settings', namespace='CELERY')

# 自动发现任务
app.autodiscover_tasks()

@app.task(bind=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

在 `your_project/__init__.py` 里添加：

```python
from .celery import app as celery_app

__all__ = ('celery_app',)
```

------

# **4️⃣ 创建 Celery 任务**

在 `your_app/tasks.py` 里定义一个 Celery 任务：

```python
from celery import shared_task

@shared_task
def example_task(message):
    print(f'执行任务: {message}')
    return f'任务完成: {message}'
```

------

# **5️⃣ 运行 Celery Worker 和 Celery Beat**

### **📌 运行 Celery Worker**

```bash
celery -A your_project worker --loglevel=info
```

### **📌 运行 Celery Beat**

```bash
celery -A your_project beat --loglevel=info
```

------

# **6️⃣ 使用 Django Admin 管理任务**

Django-Celery-Beat 提供了 **Admin 界面** 来管理定时任务：

1. **运行 Django 服务器**

   ```bash
   python manage.py runserver
   ```

2. **访问 `http://127.0.0.1:8000/admin/`**

3. **创建间隔任务**

   - 进入 `Periodic Tasks`
   - **选择 `example_task` 任务**
   - **添加间隔时间（比如每 10 分钟执行一次）**
   - **启用任务**
   - **保存**

------

# **7️⃣ 使用 API 进行任务管理**

如果你想通过 API 创建和管理任务，可以使用 Django REST Framework：

## **📌 1. 创建 API 视图**

在 `your_app/views.py` 里：

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from django_celery_beat.models import PeriodicTask, IntervalSchedule
import json
from your_app.tasks import example_task

class TaskAPI(APIView):
    """API 视图：动态创建 Celery 任务"""

    def post(self, request):
        data = request.data
        task_name = data.get("task_name", "example_task")
        interval = data.get("interval", 10)
        message = data.get("message", "默认任务")

        # 创建间隔时间（如果不存在）
        schedule, created = IntervalSchedule.objects.get_or_create(
            every=interval, period=IntervalSchedule.SECONDS
        )

        # 创建任务
        task = PeriodicTask.objects.create(
            interval=schedule,
            name=f"任务-{message}",
            task=task_name,
            args=json.dumps([message]),  # 传递参数
        )

        return Response({"status": "任务已创建", "task_id": task.id})
```

## **📌 2. 配置 URL**

在 `your_app/urls.py` 里：

```python
from django.urls import path
from .views import TaskAPI

urlpatterns = [
    path('api/tasks/', TaskAPI.as_view(), name='task-api'),
]
```

------

# **8️⃣ 通过 API 创建任务**

```bash
curl -X POST http://127.0.0.1:8000/api/tasks/ \
     -H "Content-Type: application/json" \
     -d '{
         "task_name": "your_app.tasks.example_task",
         "interval": 30,
         "message": "通过 API 创建的任务"
     }'
```

这样，Celery 就会每 **30 秒** 执行 `example_task`，并输出：

```
执行任务: 通过 API 创建的任务
```

------

# **📌 总结**

| 方案             | 方式          | 适用场景           | 是否可扩展                  |
| ---------------- | ------------- | ------------------ | --------------------------- |
| **Django Admin** | 手动管理任务  | 适用于手动调整任务 | ✅ 可管理 Cron/Interval 任务 |
| **API 方式**     | 通过 API 创建 | 适用于动态创建任务 | ✅ 可动态管理任务            |
| **Celery Beat**  | 定期执行任务  | 适用于周期任务     | ✅ 任务数据存数据库          |

------

# **🚀 你的 Celery Beat 任务管理系统已完成**

- **支持** 一次性任务、周期任务（Interval & Cron）
- **支持** Django Admin 手动管理任务
- **支持** API 方式动态创建任务
- **支持** Celery Beat 自动调度任务
- **支持** 可扩展的任务管理方式

这样，你就拥有了一个 **企业级的 Celery 任务调度管理系统** 🎯🚀

如果你只使用 **`django-celery-beat`** 来管理定时任务，并且不需要使用 **`django-celery-results`** 来存储任务结果，你可以将任务结果直接存储在 **Redis** 或自己的数据库中。

下面是一个 **完整的可扩展方案**，实现了 **对各种类型的 Celery 任务的管理**，包括周期任务（Interval）、定时任务（Cron）等。

## **📌 主要流程：**

1. **使用 `django-celery-beat`** 来管理 Celery 定时任务。
2. **使用 Redis 作为 Celery 的消息队列**（Broker），并使用你自己的数据库来存储任务执行的结果。
3. **使用 Django Admin 或 API 动态管理任务**。

------

## **1️⃣ 安装依赖**

安装以下必要的依赖：

```bash
pip install celery redis django-celery-beat djangorestframework
```

------

## **2️⃣ 配置 `settings.py`**

在 `settings.py` 配置 Celery、Redis 和 `django-celery-beat`。

```python
# settings.py

from celery.schedules import crontab

# Celery 配置
CELERY_BROKER_URL = 'redis://localhost:6379/0'  # Redis 作为消息队列
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'  # Redis 存储任务结果，或使用自己的数据库
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'

# Celery Beat 配置
INSTALLED_APPS = [
    ...,
    'django_celery_beat',  # 用于管理周期任务
]
```

然后运行数据库迁移：

```bash
python manage.py migrate django_celery_beat
```

------

## **3️⃣ 配置 Celery**

在 `your_project/celery.py` 文件中配置 Celery。

```python
import os
from celery import Celery

# 设置 Django 的默认设置模块
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'your_project.settings')

app = Celery('your_project')

# 从 settings.py 加载配置
app.config_from_object('django.conf:settings', namespace='CELERY')

# 自动发现任务
app.autodiscover_tasks()

@app.task(bind=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

在 `your_project/__init__.py` 中添加以下代码，确保 Celery 在启动时加载：

```python
from __future__ import absolute_import, unicode_literals

# 设置默认 Django 设置模块
from .celery import app as celery_app

__all__ = ('celery_app',)
```

------

## **4️⃣ 创建 Celery 任务**

在 `your_app/tasks.py` 中创建 Celery 任务。

```python
from celery import shared_task

@shared_task
def example_task(message):
    print(f'执行任务: {message}')
    return f'任务完成: {message}'

@shared_task
def another_task(message):
    print(f'执行任务: {message}')
    return f'另一个任务完成: {message}'
```

------

## **5️⃣ 在 Django Admin 中管理任务**

`django-celery-beat` 提供了 Django Admin 界面来管理周期任务（比如，定时任务、间隔任务）。

1. 启动 **Django 服务**：

```bash
python manage.py runserver
```

1. **进入 Admin 后台**：`http://127.0.0.1:8000/admin/`。
2. 创建周期任务：进入 `Periodic Tasks`，创建 **定时任务** 或 **间隔任务**。

例如，创建一个每 **10 秒** 执行一次的 `example_task`。

------

## **6️⃣ 使用 Django REST API 管理任务**

如果你希望通过 API 动态管理任务，可以使用 Django REST Framework 创建一个 API。

### 1. **创建 API 视图**

在 `your_app/views.py` 中：

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from django_celery_beat.models import PeriodicTask, IntervalSchedule, CrontabSchedule
import json
from your_app.tasks import example_task, another_task

class TaskAPI(APIView):
    """API 视图：动态创建和管理 Celery 任务"""

    def post(self, request):
        data = request.data
        task_name = data.get("task_name", "your_app.tasks.example_task")
        interval = data.get("interval", 10)  # 任务间隔，单位为秒
        message = data.get("message", "默认任务")
        schedule_type = data.get("schedule_type", "interval")  # 任务类型：interval 或 cron
        cron_expression = data.get("cron_expression", "")  # Cron 表达式

        if schedule_type == "interval":
            # 创建或获取 IntervalSchedule（周期任务间隔）
            schedule, created = IntervalSchedule.objects.get_or_create(
                every=interval, period=IntervalSchedule.SECONDS
            )
        elif schedule_type == "cron":
            # 创建 Cron 表达式任务
            schedule, created = CrontabSchedule.objects.get_or_create(
                minute=cron_expression.get("minute", "*"),
                hour=cron_expression.get("hour", "*"),
                day_of_week=cron_expression.get("day_of_week", "*"),
                day_of_month=cron_expression.get("day_of_month", "*"),
                month_of_year=cron_expression.get("month_of_year", "*"),
            )

        # 创建 PeriodicTask（周期任务）
        task = PeriodicTask.objects.create(
            interval=schedule,
            name=f"任务-{message}",
            task=task_name,
            args=json.dumps([message]),  # 任务参数
        )

        return Response({"status": "任务已创建", "task_id": task.id})
```

### 2. **配置 URL 路由**

在 `your_app/urls.py` 中配置 API 路由：

```python
from django.urls import path
from .views import TaskAPI

urlpatterns = [
    path('api/tasks/', TaskAPI.as_view(), name='task-api'),
]
```

------

## **7️⃣ 测试通过 API 创建任务**

你可以通过 API 动态创建周期任务：

### 1. **启动 Django 服务**：

```bash
python manage.py runserver
```

### 2. **通过 `POST` 请求创建任务**：

使用 `curl` 命令或 Postman 创建任务。

#### 示例：创建一个每 **30 秒** 执行一次的任务：

```bash
curl -X POST http://127.0.0.1:8000/api/tasks/ \
     -H "Content-Type: application/json" \
     -d '{
         "task_name": "your_app.tasks.example_task",
         "interval": 30,
         "message": "通过 API 创建的任务",
         "schedule_type": "interval"
     }'
```

#### 示例：创建一个使用 Cron 表达式的任务：

```bash
curl -X POST http://127.0.0.1:8000/api/tasks/ \
     -H "Content-Type: application/json" \
     -d '{
         "task_name": "your_app.tasks.another_task",
         "cron_expression": {"minute": "*/5"},
         "message": "通过 Cron 表达式创建的任务",
         "schedule_type": "cron"
     }'
```

------

## **8️⃣ 总结**

### **关键功能：**

1. **`django-celery-beat`** 用于管理周期任务（定时任务和间隔任务），支持通过 **Admin** 和 **API** 动态创建任务。
2. 通过 **Redis** 作为 Celery 的 **消息队列**（Broker），并使用你的 **自定义数据库** 来持久化存储任务结果。
3. 使用 **Django REST Framework** 创建 API，可以动态管理任务。

### **可扩展性：**

- 你可以扩展任务类型支持，如添加更多的任务参数，或者支持复杂的 `Cron` 表达式。
- API 端点支持动态创建、删除、修改任务，因此非常灵活。
- 可以根据需求将任务结果存储在自己的数据库中（不使用 `django-celery-results`），以提供自定义的数据持久化方式。

### **适用场景：**

- **周期性任务管理**：需要周期性任务的调度，支持定时任务和间隔任务。
- **自定义结果存储**：使用自定义数据库来持久化存储任务结果，而不依赖 `django-celery-results`。
- **灵活的任务管理**：通过 API 或 Admin 动态管理任务。

------

这样，你就实现了一个灵活且可扩展的 **Celery 任务调度系统**，并能通过 **Django Admin 或 REST API** 轻松管理任务！🚀

你希望在 **页面上管理 Celery 任务**，但不只是创建周期任务，而是能够在页面上配置和管理 **已编写的 Celery 任务**（即，开发者定义的任务）。

**🔹 核心目标：**

1. **开发者定义 Celery 任务**，并能在页面中选择、配置它们（而不是用户自己填写任务路径）。
2. **支持各种任务类型**：一次性任务（Ad-hoc Task）、定时任务（Periodic Task）。
3. **用户可以动态配置任务参数**，并存储到数据库。
4. **可扩展性**：未来可以支持更多任务类型。

------

## **📌 解决方案**

我们可以 **在 Django 后台或 API 提供 Celery 任务的可选列表**，然后：

- **页面上显示可用 Celery 任务列表**（从代码动态获取）。
- **用户选择任务，并输入参数，提交后创建任务**。
- **支持定时/一次性任务，并存储到数据库**。
- **Celery Worker 读取任务并执行**。

------

## **1️⃣ 获取所有 Celery 任务**

你可以动态获取 **所有定义的 Celery 任务**，而不是让用户手动输入任务名称。

### **📌 方法：遍历 Celery 已注册的任务**

Celery **会自动注册** `@shared_task` 的任务，我们可以用 `app.tasks` 获取任务列表。

```python
# your_app/utils.py
from celery import current_app

def get_all_registered_tasks():
    """ 获取所有可用的 Celery 任务 """
    return list(current_app.tasks.keys())
```

你可以在 Django API 里调用这个方法，把可用任务返回给前端页面。

------

## **2️⃣ 创建 Django API 获取任务列表**

在 `views.py` 里，提供一个 API，返回 **所有可用 Celery 任务**。

```python
# your_app/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from your_app.utils import get_all_registered_tasks

class TaskListAPI(APIView):
    """ API 视图：获取所有 Celery 任务 """

    def get(self, request):
        tasks = get_all_registered_tasks()
        return Response({"tasks": tasks})
```

### **📌 配置 Django 路由**

在 `urls.py` 里添加 API 入口：

```python
from django.urls import path
from .views import TaskListAPI

urlpatterns = [
    path('api/tasks/', TaskListAPI.as_view(), name='task-list'),
]
```

#### **📌 测试 API**

启动 Django 服务器：

```bash
python manage.py runserver
```

然后访问：

```
http://127.0.0.1:8000/api/tasks/
```

返回 JSON：

```json
{
    "tasks": [
        "your_app.tasks.example_task",
        "your_app.tasks.another_task",
        "celery.backend_cleanup"
    ]
}
```

------

## **3️⃣ 在页面选择任务并配置参数**

前端页面可以调用 `http://127.0.0.1:8000/api/tasks/`，然后让用户选择任务，并输入参数。

**示例前端表单：**

- 选择任务：`<select>` 下拉框，选择 `example_task`
- 输入参数：一个 `<input>` 让用户输入参数
- 选择执行方式：
  - **立即执行**（Ad-hoc Task）
  - **定时执行**（Scheduled Task）
  - **周期执行**（Periodic Task）

------

## **4️⃣ 用户提交任务，后端创建任务**

当用户提交任务后，Django API 需要 **根据用户选择的任务和参数，创建 Celery 任务**。

在 `views.py` 添加 API **创建任务**：

```python
import json
from django_celery_beat.models import PeriodicTask, IntervalSchedule
from rest_framework.views import APIView
from rest_framework.response import Response
from celery import current_app
from datetime import datetime, timedelta

class TaskCreateAPI(APIView):
    """ API 视图：用户动态创建 Celery 任务 """

    def post(self, request):
        """
        请求格式：
        {
            "task_name": "your_app.tasks.example_task",
            "params": ["Hello, Celery!"],  # 任务参数
            "execution_type": "adhoc" | "scheduled" | "periodic",
            "run_at": "2025-04-01 10:00:00",  # 仅对 scheduled 任务有效
            "interval_seconds": 60  # 仅对 periodic 任务有效
        }
        """
        data = request.data
        task_name = data.get("task_name")
        params = data.get("params", [])
        execution_type = data.get("execution_type")

        if task_name not in current_app.tasks:
            return Response({"error": "任务不存在"}, status=400)

        # **1️⃣ 立即执行的任务**
        if execution_type == "adhoc":
            result = current_app.send_task(task_name, args=params)
            return Response({"status": "任务已提交", "task_id": result.id})

        # **2️⃣ 定时执行的任务**
        elif execution_type == "scheduled":
            run_at = datetime.strptime(data.get("run_at"), "%Y-%m-%d %H:%M:%S")
            result = current_app.send_task(task_name, args=params, eta=run_at)
            return Response({"status": "任务已定时", "task_id": result.id})

        # **3️⃣ 周期执行的任务**
        elif execution_type == "periodic":
            interval, _ = IntervalSchedule.objects.get_or_create(
                every=data.get("interval_seconds", 60),
                period=IntervalSchedule.SECONDS
            )
            task = PeriodicTask.objects.create(
                interval=interval,
                name=f"{task_name}-{datetime.now()}",
                task=task_name,
                args=json.dumps(params),
            )
            return Response({"status": "周期任务已创建", "task_id": task.id})

        return Response({"error": "未知的执行类型"}, status=400)
```

------

## **5️⃣ 配置 Django 路由**

在 `urls.py` 里添加创建任务的 API 入口：

```python
from django.urls import path
from .views import TaskCreateAPI

urlpatterns = [
    path('api/create-task/', TaskCreateAPI.as_view(), name='create-task'),
]
```

------

## **6️⃣ 前端调用 API 提交任务**

用户选择任务后，前端提交：

```bash
curl -X POST http://127.0.0.1:8000/api/create-task/ \
     -H "Content-Type: application/json" \
     -d '{
         "task_name": "your_app.tasks.example_task",
         "params": ["Hello, Celery!"],
         "execution_type": "adhoc"
     }'
```

返回：

```json
{
    "status": "任务已提交",
    "task_id": "4c5e7a32-ef4e-4c4e-9b3f-7e32d813e676"
}
```

------

## **🎯 总结**

1. **获取所有 Celery 任务**，动态显示给用户选择 ✅
2. **用户输入参数、选择执行方式**（立即执行/定时执行/周期执行）✅
3. **Django API 处理任务**：
   - **立即执行**（Ad-hoc Task）🚀
   - **定时执行**（Scheduled Task）⏰
   - **周期任务**（Periodic Task，存入 `django-celery-beat`）📅
4. **Celery Worker 执行任务** 🎯

------

💡 **这样，你的 Celery 任务管理系统变得更灵活，可以在页面上自由配置任务！** 🚀

在 Django 中实现一个 Celery 任务管理页面，主要包括以下功能：

1. **查看任务列表**（包括运行中的、等待中的、失败的任务）。
2. **获取任务详情**（任务参数、状态、结果等）。
3. **取消/终止任务**。
4. **重新运行任务**（失败的任务或指定任务）。
5. **清除已完成或失败的任务**。

## **后端实现逻辑**

使用 **Celery 提供的 API** 获取任务信息，并使用 Django 的 **DRF（Django REST Framework）** 设计 API 供前端调用。

------

## **1. Celery 配置**

确保 Celery 已在 Django 项目中正确配置，并且 Celery Beat 用于周期性任务管理：

```python
# settings.py
CELERY_BROKER_URL = "redis://localhost:6379/0"  # 或其他消息队列
CELERY_RESULT_BACKEND = "redis://localhost:6379/0"
```

------

## **2. Celery 任务管理 API**

创建 Django API 视图，管理 Celery 任务。

```python
from celery.result import AsyncResult
from django_celery_results.models import TaskResult
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from myapp.tasks import sample_task  # 你的任务

# 获取所有任务
class TaskListView(APIView):
    def get(self, request):
        tasks = TaskResult.objects.all().values("task_id", "status", "date_done", "result")
        return Response({"tasks": list(tasks)}, status=status.HTTP_200_OK)


# 获取单个任务详情
class TaskDetailView(APIView):
    def get(self, request, task_id):
        result = AsyncResult(task_id)
        return Response({
            "task_id": task_id,
            "status": result.status,
            "result": result.result if result.ready() else None
        }, status=status.HTTP_200_OK)


# 终止任务
class TaskRevokeView(APIView):
    def post(self, request, task_id):
        result = AsyncResult(task_id)
        result.revoke(terminate=True)
        return Response({"message": f"Task {task_id} revoked"}, status=status.HTTP_200_OK)


# 重新执行任务
class TaskRetryView(APIView):
    def post(self, request, task_id):
        result = AsyncResult(task_id)
        if result.failed():
            new_task = sample_task.apply_async(args=result.args, kwargs=result.kwargs)
            return Response({"message": "Task retried", "new_task_id": new_task.id}, status=status.HTTP_200_OK)
        return Response({"error": "Task is not failed"}, status=status.HTTP_400_BAD_REQUEST)


# 清除任务
class TaskPurgeView(APIView):
    def delete(self, request):
        TaskResult.objects.all().delete()
        return Response({"message": "All tasks cleared"}, status=status.HTTP_200_OK)
```

------

## **3. 注册 API 路由**

在 `urls.py` 中添加 API 路由：

```python
from django.urls import path
from .views import TaskListView, TaskDetailView, TaskRevokeView, TaskRetryView, TaskPurgeView

urlpatterns = [
    path("tasks/", TaskListView.as_view(), name="task-list"),
    path("tasks/<str:task_id>/", TaskDetailView.as_view(), name="task-detail"),
    path("tasks/<str:task_id>/revoke/", TaskRevokeView.as_view(), name="task-revoke"),
    path("tasks/<str:task_id>/retry/", TaskRetryView.as_view(), name="task-retry"),
    path("tasks/purge/", TaskPurgeView.as_view(), name="task-purge"),
]
```

------

## **4. 创建 Celery 任务**

示例 Celery 任务：

```python
from celery import shared_task
import time

@shared_task(bind=True)
def sample_task(self, x, y):
    time.sleep(5)  # 模拟长时间任务
    return x + y
```

------

## **5. 测试 API**

启动 Celery Worker：

```sh
celery -A myproject worker --loglevel=info
```

测试创建任务：

```sh
curl -X POST http://localhost:8000/tasks/ -H "Content-Type: application/json"
```

测试查看任务列表：

```sh
curl http://localhost:8000/tasks/
```

测试终止任务：

```sh
curl -X POST http://localhost:8000/tasks/<task_id>/revoke/
```

------

## **总结**

这样，我们就实现了一个 Celery 任务管理系统，支持：

- 任务查询（进行中、已完成、失败）。
- 任务详情查看。
- 任务终止。
- 任务重试。
- 任务清理。

前端可以调用这些 API，实现任务管理界面。

如果需要 **管理 Celery 任务的队列、排队、分发等**，可以使用 **Celery 的队列、优先级、路由等机制**，并扩展 API 以提供队列管理功能。以下是详细的实现方式：

------

## **1. 任务队列管理**

Celery 允许创建多个队列，并将不同任务分配到不同队列中。

### **（1）定义不同的队列**

在 `settings.py` 配置多个队列：

```python
CELERY_BROKER_URL = "redis://localhost:6379/0"
CELERY_RESULT_BACKEND = "redis://localhost:6379/0"

# 定义队列
CELERY_TASK_QUEUES = {
    "default": {"exchange": "default", "routing_key": "default"},
    "high_priority": {"exchange": "high", "routing_key": "high"},
    "low_priority": {"exchange": "low", "routing_key": "low"},
}
```

### **（2）指定任务使用特定队列**

在 `tasks.py` 里，设置任务默认队列：

```python
from celery import shared_task

@shared_task(queue="high_priority")
def high_priority_task(x, y):
    return x + y

@shared_task(queue="low_priority")
def low_priority_task(x, y):
    return x - y
```

### **（3）启动不同的 Celery Worker 监听不同的队列**

```sh
# 启动默认队列的 Worker
celery -A myproject worker --loglevel=info -Q default

# 启动高优先级队列的 Worker
celery -A myproject worker --loglevel=info -Q high_priority

# 启动低优先级队列的 Worker
celery -A myproject worker --loglevel=info -Q low_priority
```

------

## **2. 任务路由**

你可以通过 Celery 路由规则 **自动分发任务到指定队列**。

### **（1）在 `celery.py` 中配置任务路由**

```python
CELERY_TASK_ROUTES = {
    "myapp.tasks.high_priority_task": {"queue": "high_priority"},
    "myapp.tasks.low_priority_task": {"queue": "low_priority"},
}
```

------

## **3. API 扩展：动态分配任务到不同队列**

在 Django API 里，添加管理队列的接口。

### **（1）动态创建任务到不同队列**

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from myapp.tasks import high_priority_task, low_priority_task
from celery import current_app

# 提交任务到不同的队列
class TaskCreateView(APIView):
    def post(self, request):
        queue = request.data.get("queue", "default")
        x = request.data.get("x", 1)
        y = request.data.get("y", 2)

        if queue == "high_priority":
            task = high_priority_task.apply_async(args=[x, y], queue="high_priority")
        elif queue == "low_priority":
            task = low_priority_task.apply_async(args=[x, y], queue="low_priority")
        else:
            return Response({"error": "Invalid queue"}, status=status.HTTP_400_BAD_REQUEST)

        return Response({"task_id": task.id, "queue": queue}, status=status.HTTP_201_CREATED)
```

### **（2）获取所有队列信息**

```python
class QueueListView(APIView):
    def get(self, request):
        inspector = current_app.control.inspect()
        active_queues = inspector.active_queues()
        return Response({"queues": active_queues}, status=status.HTTP_200_OK)
```

### **（3）清空特定队列**

```python
class QueuePurgeView(APIView):
    def delete(self, request, queue_name):
        inspector = current_app.control.inspect()
        inspector.purge(queue=queue_name)
        return Response({"message": f"Queue {queue_name} purged"}, status=status.HTTP_200_OK)
```

------

## **4. 任务优先级**

Celery 支持任务的优先级（但仅适用于 RabbitMQ）。

```python
@shared_task(queue="default")
def priority_task(x, y, priority=5):
    return x + y
```

发送任务时设置优先级：

```python
priority_task.apply_async(args=[1, 2], queue="default", priority=0)  # 最高优先级
priority_task.apply_async(args=[3, 4], queue="default", priority=9)  # 最低优先级
```

------

## **5. 任务调度**

Celery Beat 可以定时执行任务：

```python
@shared_task(queue="default")
def scheduled_task():
    print("This task runs every minute!")
```

在 `celery.py` 配置 Celery Beat：

```python
CELERY_BEAT_SCHEDULE = {
    "run_scheduled_task": {
        "task": "myapp.tasks.scheduled_task",
        "schedule": crontab(minute="*/1"),
    },
}
```

启动 Celery Beat：

```sh
celery -A myproject beat --loglevel=info
```

------

## **6. 任务分发管理**

可以动态管理任务并控制 Worker：

```python
from celery import current_app

class WorkerControlView(APIView):
    def post(self, request, action):
        inspector = current_app.control.inspect()

        if action == "pause":
            inspector.suspend()
            return Response({"message": "Workers paused"}, status=status.HTTP_200_OK)
        elif action == "resume":
            inspector.resume()
            return Response({"message": "Workers resumed"}, status=status.HTTP_200_OK)
        elif action == "terminate":
            inspector.shutdown()
            return Response({"message": "Workers terminated"}, status=status.HTTP_200_OK)
        else:
            return Response({"error": "Invalid action"}, status=status.HTTP_400_BAD_REQUEST)
```

前端可以调用：

```sh
curl -X POST http://localhost:8000/workers/pause/  # 暂停 Worker
curl -X POST http://localhost:8000/workers/resume/  # 恢复 Worker
curl -X POST http://localhost:8000/workers/terminate/  # 终止 Worker
```

------

## **总结**

✅ **任务队列管理**

- 可以动态分配任务到不同队列。
- 支持查看所有队列状态。
- 允许清空某个队列的任务。

✅ **任务优先级**

- RabbitMQ 支持任务优先级队列。

✅ **任务调度**

- Celery Beat 可用于定时任务。

✅ **Worker 管理**

- 可以暂停、恢复或终止 Worker 进程。

这样，整个 Celery 任务管理系统就具备了 **任务监控、调度、分发、队列管理、任务终止、Worker 维护** 等完整功能！ 🚀

在 Celery 任务队列中，默认情况下任务是按 **FIFO（先进先出）** 方式执行的，**Redis** 作为消息队列（Broker）时，任务是按照队列顺序存储的。但 Celery 本身**不支持直接调整任务的队列顺序**，要实现 **"更改任务的排队位置"**，可以使用以下几种方案：

------

## **方案 1：重新提交任务（推荐）**

### **思路**

1. **找到目标任务**（即需要调整顺序的任务）。
2. **取消任务（revoke）**，但不终止正在执行的 Worker 进程。
3. **重新提交任务**，让它排到队列最前面。

### **实现**

```python
from celery.result import AsyncResult
from celery import current_app
from django_celery_results.models import TaskResult
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from myapp.tasks import sample_task  # 需要重新提交的任务

class TaskReorderView(APIView):
    def post(self, request):
        task_id = request.data.get("task_id")

        # 获取任务状态
        result = AsyncResult(task_id)

        if result.state not in ["PENDING", "RECEIVED"]:  # 只允许调整未执行的任务
            return Response({"error": "Task is already running or completed"}, status=status.HTTP_400_BAD_REQUEST)

        # 获取原任务参数（如果任务存储了参数）
        original_task = TaskResult.objects.filter(task_id=task_id).first()
        if not original_task:
            return Response({"error": "Task not found"}, status=status.HTTP_404_NOT_FOUND)

        # 取消原任务
        result.revoke(terminate=False)

        # 重新提交任务到队列最前面
        new_task = sample_task.apply_async(args=original_task.args, kwargs=original_task.kwargs, queue=original_task.task_queue, priority=0)

        return Response({"message": "Task reordered", "new_task_id": new_task.id}, status=status.HTTP_200_OK)
```

### **说明**

- `revoke(terminate=False)` 只取消任务，但不会影响 Worker 进程。
- `apply_async(..., priority=0)` 重新提交任务，并设定优先级为最高（RabbitMQ 支持）。
- 适用于 Redis/RabbitMQ，但 **任务 ID 会变化**，需要通知用户新的任务 ID。

------

## **方案 2：使用自定义任务队列**

### **思路**

1. 维护一个 **任务列表**（比如存储在 **Redis** 或 **数据库** 中）。
2. 提交任务时，不直接放入 Celery，而是先入自定义队列。
3. **用户可调整队列顺序**，然后再手动触发 Celery 执行。

### **实现**

使用 **Redis** 存储任务队列：

```python
import redis
import json
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from myapp.tasks import sample_task

redis_client = redis.StrictRedis(host="localhost", port=6379, db=1)

TASK_QUEUE_KEY = "custom_task_queue"

class TaskQueueView(APIView):
    """ 提交任务到 Redis 队列 """
    def post(self, request):
        task_data = {
            "x": request.data.get("x", 1),
            "y": request.data.get("y", 2),
            "priority": request.data.get("priority", 5)
        }
        redis_client.rpush(TASK_QUEUE_KEY, json.dumps(task_data))
        return Response({"message": "Task added to queue"}, status=status.HTTP_201_CREATED)

class TaskReorderView(APIView):
    """ 调整任务顺序 """
    def post(self, request):
        task_index = request.data.get("index")  # 需要调整的任务位置
        new_index = request.data.get("new_index")  # 目标位置

        queue = redis_client.lrange(TASK_QUEUE_KEY, 0, -1)
        if task_index >= len(queue) or new_index >= len(queue):
            return Response({"error": "Invalid index"}, status=status.HTTP_400_BAD_REQUEST)

        # 交换任务顺序
        task = queue.pop(task_index)
        queue.insert(new_index, task)

        # 更新 Redis 队列
        redis_client.delete(TASK_QUEUE_KEY)
        for item in queue:
            redis_client.rpush(TASK_QUEUE_KEY, item)

        return Response({"message": "Task order updated"}, status=status.HTTP_200_OK)

class TaskWorkerView(APIView):
    """ Worker 从 Redis 队列拉取任务并提交到 Celery """
    def post(self, request):
        task_data = redis_client.lpop(TASK_QUEUE_KEY)
        if not task_data:
            return Response({"error": "No tasks in queue"}, status=status.HTTP_400_BAD_REQUEST)

        task = json.loads(task_data)
        new_task = sample_task.apply_async(args=[task["x"], task["y"]], priority=task["priority"])

        return Response({"message": "Task submitted", "task_id": new_task.id}, status=status.HTTP_200_OK)
```

### **流程**

1. `TaskQueueView`: 用户提交任务，任务存入 Redis 队列。
2. `TaskReorderView`: 允许用户调整任务在 Redis 队列中的顺序。
3. `TaskWorkerView`: 由后台进程**主动拉取任务**并提交给 Celery。

### **适用场景**

- 适用于 Redis 作为 Broker，且任务**需要动态调整**执行顺序的情况。
- 适用于**任务必须严格按照调整后顺序执行**的场景。

------

## **方案 3：任务优先级调整（适用于 RabbitMQ）**

**RabbitMQ** 支持**任务优先级队列**，可让高优先级任务先执行。

### **步骤**

1. **设置队列支持优先级**（`x-max-priority`）。
2. **任务重新提交时提升优先级**。

### **Celery 配置**

```python
CELERY_TASK_QUEUES = {
    "default": {
        "exchange": "default",
        "routing_key": "default",
        "queue_arguments": {"x-max-priority": 10},  # 最高 10 级
    }
}
```

### **任务重新提交时调整优先级**

```python
from myapp.tasks import sample_task
from rest_framework.views import APIView
from rest_framework.response import Response

class TaskPriorityView(APIView):
    def post(self, request):
        task_id = request.data.get("task_id")
        new_priority = request.data.get("priority", 5)

        result = AsyncResult(task_id)
        if result.state not in ["PENDING", "RECEIVED"]:
            return Response({"error": "Task is already running or completed"}, status=400)

        # 取消原任务
        result.revoke(terminate=False)

        # 重新提交任务，提升优先级
        new_task = sample_task.apply_async(args=result.args, kwargs=result.kwargs, priority=new_priority)

        return Response({"message": "Task priority updated", "new_task_id": new_task.id}, status=200)
```

### **适用场景**

- **适用于 RabbitMQ**，但 Redis 不支持 `priority` 参数。

------

## **总结**

| 方案                                   | 适用场景                      | 优点                          | 缺点                 |
| -------------------------------------- | ----------------------------- | ----------------------------- | -------------------- |
| **方案 1：重新提交任务（推荐）**       | 适用于 **大多数 Celery 环境** | 实现简单，通用性强            | 任务 ID 会变化       |
| **方案 2：自定义队列管理（Redis）**    | 任务需要**严格按照顺序调整**  | 任务顺序可随意调整            | 需要额外 Worker 处理 |
| **方案 3：任务优先级调整（RabbitMQ）** | RabbitMQ 作为 Broker          | 不影响任务 ID，直接调整优先级 | 仅适用于 RabbitMQ    |

✅ **如果你使用 Redis**，建议 **方案 1 或 2**。
 ✅ **如果你使用 RabbitMQ**，可以使用 **方案 3**（任务优先级）。

如果你使用的是 **Redis 作为 Celery 的 Broker**，并且希望**更简单**地调整任务顺序，可以使用 **Redis 的 LPUSH/RPUSH 重新插入任务**，避免撤销和重新提交任务带来的 ID 变化。

------

## **最简单方案：Redis 直接调整任务队列**

### **思路**

1. **任务入队时使用 Redis List 存储任务 ID**（而不是让 Celery 直接执行）。
2. **用户可以调整 Redis List 里的顺序**（LPUSH、LINSERT 操作）。
3. **Worker 从 Redis List 拉取任务并提交到 Celery**。

------

## **实现步骤**

### **（1）任务入队（推送到 Redis 队列）**

```python
import redis
import json
from myapp.tasks import sample_task  # 你的 Celery 任务

redis_client = redis.StrictRedis(host="localhost", port=6379, db=1)
TASK_QUEUE_KEY = "custom_task_queue"

def add_task_to_queue(x, y):
    """ 将任务添加到 Redis 队列 """
    task_data = json.dumps({"x": x, "y": y})
    redis_client.rpush(TASK_QUEUE_KEY, task_data)  # 任务排到队列末尾
    return {"message": "Task added to queue"}
```

------

### **（2）调整任务顺序**

```python
def reorder_task(old_index, new_index):
    """ 调整任务顺序 """
    queue = redis_client.lrange(TASK_QUEUE_KEY, 0, -1)
    if old_index >= len(queue) or new_index >= len(queue):
        return {"error": "Invalid index"}

    task = queue.pop(old_index)  # 取出任务
    queue.insert(new_index, task)  # 插入到新位置

    # 更新 Redis 队列
    redis_client.delete(TASK_QUEUE_KEY)
    for item in queue:
        redis_client.rpush(TASK_QUEUE_KEY, item)

    return {"message": "Task reordered"}
```

------

### **（3）Worker 拉取任务并提交 Celery**

```python
def process_task():
    """ 从 Redis 队列拉取任务并交给 Celery """
    task_data = redis_client.lpop(TASK_QUEUE_KEY)
    if not task_data:
        return {"error": "No tasks in queue"}

    task = json.loads(task_data)
    celery_task = sample_task.apply_async(args=[task["x"], task["y"]])
    return {"message": "Task submitted", "task_id": celery_task.id}
```

------

## **最终效果**

1. **用户提交任务** → `rpush` 排队。
2. **用户调整顺序** → `lrange` 获取队列，调整顺序后 `delete`+`rpush` 覆盖队列。
3. **后台 Worker 执行任务** → `lpop` 获取任务并提交到 Celery。

------

## **优点**

✅ **任务顺序可随时调整**（不需要撤销任务）。
 ✅ **任务 ID 不变**（任务数据存 Redis，直到 Worker 取出执行）。
 ✅ **无需修改 Celery 配置**，只是在 Celery 之前加了一层 Redis 任务管理。

------

## **适用场景**

- **任务排队时间较长**，需要调整顺序的情况（如批量任务）。
- **想要手动控制任务执行时机**，而不是 Celery 自动处理。

你可以结合 **Django API** 让前端可以拖拽调整任务顺序 🚀。

如果你希望 **Redis 队列在空闲时自动执行任务**，但**排队时可以手动调整顺序**，可以使用 **两种模式**：

1. **任务少时**，任务直接交给 Celery **自动执行**。
2. **任务多时**，任务先进入 **Redis 队列**，可**手动调整顺序**，然后由 Worker 拉取执行。

------

## **方案概述**

- 任务少时（队列长度 = 0）：直接 `apply_async()` **提交任务**，让 Celery 立刻执行。
- 任务多时（队列长度 > 0）：任务 **先进入 Redis 队列**，可手动调整顺序，然后由 Worker 依次提交到 Celery。
- **手动调整任务顺序**：可以**前端提供接口**，调整任务在 Redis 队列中的位置。

------

## **实现步骤**

### **（1）任务提交逻辑**

```python
import redis
import json
from myapp.tasks import sample_task  # 你的 Celery 任务

redis_client = redis.StrictRedis(host="localhost", port=6379, db=1)
TASK_QUEUE_KEY = "custom_task_queue"

def add_task(x, y):
    """ 
    任务提交：
    - 如果队列为空，直接执行任务
    - 否则，进入 Redis 队列（可手动调整）
    """
    queue_length = redis_client.llen(TASK_QUEUE_KEY)

    task_data = json.dumps({"x": x, "y": y})

    if queue_length == 0:
        # 队列为空，直接交给 Celery 执行
        task = sample_task.apply_async(args=[x, y])
        return {"message": "Task executed immediately", "task_id": task.id}
    else:
        # 队列已有任务，先排队
        redis_client.rpush(TASK_QUEUE_KEY, task_data)
        return {"message": "Task added to queue"}
```

✅ **任务少时直接执行**，任务多时进队列等待。

------

### **（2）手动调整任务顺序**

```python
def reorder_task(old_index, new_index):
    """ 调整任务顺序 """
    queue = redis_client.lrange(TASK_QUEUE_KEY, 0, -1)
    if old_index >= len(queue) or new_index >= len(queue):
        return {"error": "Invalid index"}

    task = queue.pop(old_index)  # 取出任务
    queue.insert(new_index, task)  # 插入到新位置

    # 更新 Redis 队列
    redis_client.delete(TASK_QUEUE_KEY)
    for item in queue:
        redis_client.rpush(TASK_QUEUE_KEY, item)

    return {"message": "Task reordered"}
```

✅ **允许用户拖拽调整任务顺序**（类似前端拖拽队列排序）。

------

### **（3）Worker 处理排队任务**

```python
import time

def process_queued_tasks():
    """ 循环检查 Redis 队列，执行排队任务 """
    while True:
        queue_length = redis_client.llen(TASK_QUEUE_KEY)

        if queue_length > 0:
            task_data = redis_client.lpop(TASK_QUEUE_KEY)
            if task_data:
                task = json.loads(task_data)
                sample_task.apply_async(args=[task["x"], task["y"]])
                print("Task submitted from queue:", task)
        
        time.sleep(2)  # 每 2 秒检查一次队列
```

✅ **如果队列有任务，依次提交到 Celery 处理**。

------

### **（4）启动 Worker 线程**

可以在 **Django 启动时**，或者作为一个 **独立进程** 运行这个 Worker：

```bash
python manage.py run_task_worker
```

或者，创建 Django 命令：

```python
from django.core.management.base import BaseCommand
from myapp.utils import process_queued_tasks

class Command(BaseCommand):
    help = "Process queued Celery tasks"

    def handle(self, *args, **kwargs):
        process_queued_tasks()
```

✅ **这样，Redis 任务队列会一直运行**，自动拉取任务执行。

------

## **最终实现效果**

| 操作                     | 处理逻辑                              |
| ------------------------ | ------------------------------------- |
| **任务少时（队列空）**   | 任务直接交给 Celery 立即执行          |
| **任务多时（队列非空）** | 任务进入 Redis 队列，可手动调整顺序   |
| **手动调整任务**         | 用户可调整任务排队顺序                |
| **后台自动处理任务**     | Worker 定时检查 Redis，拉取任务并执行 |

------

## **优点**

✅ **任务顺序可随时调整**（任务排队时可手动调整）。
 ✅ **无需撤销任务**，任务在 Redis 中调整，而不是 Celery 队列中。
 ✅ **自动任务管理**，空闲时自动执行任务，排队时可手动调整顺序。

这样你就可以 **在 Celery 任务排队的情况下**，随时调整任务顺序，并且 **不影响任务 ID** 🎯！

下面是完整的 Django + Redis + Celery **任务管理系统**后端代码，实现了以下功能：

1. **任务直接执行**（如果队列为空）。
2. **任务排队**（如果已有任务在排队，可调整顺序）。
3. **手动调整任务顺序**（可前端操作）。
4. **后台 Worker 自动执行任务**（从 Redis 队列拉取任务并提交 Celery）。

------

## **1. 安装依赖**

确保已安装 Celery、Redis 和 Django：

```bash
pip install celery redis django django-celery-results
```

------

## **2. 配置 Celery**

### **在 `settings.py` 配置 Celery**

```python
# Redis 作为 Celery Broker
CELERY_BROKER_URL = "redis://localhost:6379/0"

# 存储 Celery 任务结果（可选）
CELERY_RESULT_BACKEND = "django-db"
```

------

## **3. 配置 `celery.py`**

创建 `myproject/celery.py`：

```python
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")

app = Celery("myproject")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

在 `__init__.py` 里加上：

```python
from .celery import app as celery_app

__all__ = ("celery_app",)
```

------

## **4. 创建 Celery 任务**

创建 `tasks.py`：

```python
from celery import shared_task
import time

@shared_task
def sample_task(x, y):
    """Celery 任务：计算 x + y"""
    time.sleep(5)  # 模拟耗时任务
    return x + y
```

------

## **5. 任务队列管理（Redis 操作）**

创建 `queue_manager.py`：

```python
import redis
import json
from myapp.tasks import sample_task

redis_client = redis.StrictRedis(host="localhost", port=6379, db=1)
TASK_QUEUE_KEY = "custom_task_queue"

def add_task(x, y):
    """ 任务提交：
    - 队列为空，直接执行
    - 队列非空，进入 Redis 等待 """
    queue_length = redis_client.llen(TASK_QUEUE_KEY)

    task_data = json.dumps({"x": x, "y": y})

    if queue_length == 0:
        # 队列为空，直接执行任务
        task = sample_task.apply_async(args=[x, y])
        return {"message": "Task executed immediately", "task_id": task.id}
    else:
        # 队列非空，加入等待队列
        redis_client.rpush(TASK_QUEUE_KEY, task_data)
        return {"message": "Task added to queue"}

def reorder_task(old_index, new_index):
    """ 调整任务顺序 """
    queue = redis_client.lrange(TASK_QUEUE_KEY, 0, -1)
    if old_index >= len(queue) or new_index >= len(queue):
        return {"error": "Invalid index"}

    task = queue.pop(old_index)  # 取出任务
    queue.insert(new_index, task)  # 插入新位置

    # 重新写入 Redis 队列
    redis_client.delete(TASK_QUEUE_KEY)
    for item in queue:
        redis_client.rpush(TASK_QUEUE_KEY, item)

    return {"message": "Task reordered"}

def process_queued_tasks():
    """ 处理 Redis 队列中的任务 """
    while True:
        queue_length = redis_client.llen(TASK_QUEUE_KEY)

        if queue_length > 0:
            task_data = redis_client.lpop(TASK_QUEUE_KEY)
            if task_data:
                task = json.loads(task_data)
                sample_task.apply_async(args=[task["x"], task["y"]])
                print("Task submitted from queue:", task)

        time.sleep(2)  # 每 2 秒检查一次队列
```

------

## **6. Django API（任务管理接口）**

在 `views.py` 里：

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from myapp.queue_manager import add_task, reorder_task

class TaskSubmitView(APIView):
    """ 提交任务 """
    def post(self, request):
        x = request.data.get("x", 1)
        y = request.data.get("y", 2)
        result = add_task(x, y)
        return Response(result, status=status.HTTP_200_OK)

class TaskReorderView(APIView):
    """ 调整任务顺序 """
    def post(self, request):
        old_index = request.data.get("old_index")
        new_index = request.data.get("new_index")
        result = reorder_task(old_index, new_index)
        return Response(result, status=status.HTTP_200_OK)
```

------

## **7. Django 路由**

在 `urls.py` 添加：

```python
from django.urls import path
from myapp.views import TaskSubmitView, TaskReorderView

urlpatterns = [
    path("task/submit/", TaskSubmitView.as_view(), name="task_submit"),
    path("task/reorder/", TaskReorderView.as_view(), name="task_reorder"),
]
```

------

## **8. 运行 Celery Worker**

```bash
celery -A myproject worker --loglevel=info
```

------

## **9. 启动后台 Worker 处理 Redis 队列**

创建 Django 命令 `myapp/management/commands/process_queue.py`：

```python
from django.core.management.base import BaseCommand
from myapp.queue_manager import process_queued_tasks

class Command(BaseCommand):
    help = "Process queued Celery tasks"

    def handle(self, *args, **kwargs):
        process_queued_tasks()
```

然后运行：

```bash
python manage.py process_queue
```

------

## **最终效果**

| 操作                     | 结果                                        |
| ------------------------ | ------------------------------------------- |
| **任务少时**（队列为空） | 直接交给 Celery 立即执行                    |
| **任务多时**（队列非空） | 任务进入 Redis 等待，可手动调整顺序         |
| **调整任务顺序**         | 可前端提供 UI 拖拽任务顺序                  |
| **Worker 自动处理任务**  | 后台 Worker 检查 Redis，并提交任务到 Celery |

------

## **总结**

✅ **任务少时立即执行，任务多时进入 Redis 队列**
 ✅ **支持手动调整任务顺序**（前端可调用 API 拖拽排序）
 ✅ **后台自动执行任务**（避免任务一直堆积）

这样你就拥有了一个 **可控的 Celery 任务队列管理系统** 🚀！

在使用 Redis 作为 Celery Broker 时，内置的任务优先级支持是有限的。也就是说，即使在调用任务时传递了 priority 参数，这个参数通常不会被 Redis Broker 实际采用，因为 Redis 本身并不支持基于优先级的队列排序。

------

### **1. 内置优先级支持**

- **RabbitMQ**：Celery 在使用 RabbitMQ 作为 Broker 时支持任务优先级，你可以在调用任务时设置 priority 参数，Broker 会根据设定的优先级调度任务。
- **Redis**：Redis 作为 Broker 时，虽然在 API 层面上可以传递 priority 参数，但它不会对任务的调度顺序产生影响，任务还是按照 FIFO（先进先出）的顺序处理。

------

### **2. 模拟任务优先级的方案**

如果你需要在 Redis 环境下实现类似优先级的调度，可以考虑以下几种方案：

#### **方案 1：使用多个队列**

你可以为不同“优先级”的任务设置不同的队列。比如：

- 定义两个队列：`high_priority` 和 `low_priority`
- 根据任务的紧急程度，将任务发送到对应队列
- 在启动 Worker 时，让 Worker 同时监听这两个队列，并通过 Worker 的启动参数来设定队列的消费顺序（例如，让 Worker 优先消费 `high_priority` 队列中的任务）

示例代码：

```python
from celery import shared_task

@shared_task(queue="high_priority")
def urgent_task(x, y):
    return x + y

@shared_task(queue="low_priority")
def normal_task(x, y):
    return x - y
```

启动 Worker 时可以指定队列的顺序，例如：

```bash
celery -A myproject worker --loglevel=info -Q high_priority,low_priority
```

这样 Worker 会优先处理 `high_priority` 队列中的任务。

------

#### **方案 2：自定义 Redis 队列管理**

如果你希望在任务提交后能动态调整任务顺序，可以采用自定义的 Redis 队列管理方案：

- **任务入队**时，将任务数据存入 Redis 的 List 中（而不是直接提交到 Celery）。
- 当队列中存在任务时，你可以通过接口调整 Redis List 中任务的顺序。
- 后台进程或 Worker 定时检查 Redis List，并根据调整后的顺序逐个将任务提交给 Celery 执行。

这种方式可以让你在任务等待期间，自由调整任务的顺序，但实现上需要额外编写代码来管理 Redis List。

------

### **3. broker_transport_options 与优先级**

在 Celery 的配置中，可以通过 `broker_transport_options` 设置一些 Redis 相关的参数，例如 `visibility_timeout` 和 `max_connections`。不过，关于任务优先级，Redis Broker 并不支持原生的优先级功能，所以即使在 `broker_transport_options` 中设置了诸如 `"queue_order_strategy": "priority"`（某些旧版本或特定实现中可能存在该选项），也不会像 RabbitMQ 那样实现真正的任务优先级调度。

示例配置（注意这只是部分配置，优先级部分在 Redis 下无效）：

```python
CELERY_BROKER_URL = "redis://localhost:6379/0"
CELERY_BROKER_TRANSPORT_OPTIONS = {
    "visibility_timeout": 3600,
    "max_connections": 10,
    # "queue_order_strategy": "priority"  # 在 Redis 中此配置通常不生效
}
```

------

### **总结**

- **直接使用优先级参数**：在 Redis Broker 下传递 `priority` 参数不会改变任务的执行顺序。
- **模拟优先级调度**：可以通过设置多个队列或自定义 Redis 队列管理的方式来实现类似的效果。
- **建议**：如果任务优先级对你非常重要，可以考虑使用支持优先级的 Broker（如 RabbitMQ），或者实现自定义的队列管理逻辑。

通过以上方式，你可以在 Redis 环境下“模拟”任务优先级，而不是依赖 Redis 本身对优先级的支持。
