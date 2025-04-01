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
