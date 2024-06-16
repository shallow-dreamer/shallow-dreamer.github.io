---
layout:     post
title:      "状态机和bulk_update"
subtitle:   ""
date:       2024-06-14
author:     " Shallow Dreamer"
header-img: "img/post-bg-js-version.jpg"
tags:
    - django
---

为了实现更复杂的流程状态机，包括撤回、驳回、并行审批和串行审批等状态，我们需要扩展模型和状态转换逻辑。下面是详细的实现步骤。

### 1. 定义复杂的状态和转换

我们需要为任务模型定义更多的状态和状态转换逻辑。

```python
from django.db import models

class Task(models.Model):
    STATUS_PENDING = 'pending'
    STATUS_IN_PROGRESS = 'in_progress'
    STATUS_COMPLETED = 'completed'
    STATUS_CANCELLED = 'cancelled'
    STATUS_REJECTED = 'rejected'
    STATUS_APPROVED = 'approved'
    STATUS_WITHDRAWN = 'withdrawn'

    STATUS_CHOICES = [
        (STATUS_PENDING, 'Pending'),
        (STATUS_IN_PROGRESS, 'In Progress'),
        (STATUS_COMPLETED, 'Completed'),
        (STATUS_CANCELLED, 'Cancelled'),
        (STATUS_REJECTED, 'Rejected'),
        (STATUS_APPROVED, 'Approved'),
        (STATUS_WITHDRAWN, 'Withdrawn'),
    ]

    APPROVAL_SERIAL = 'serial'
    APPROVAL_PARALLEL = 'parallel'

    APPROVAL_CHOICES = [
        (APPROVAL_SERIAL, 'Serial Approval'),
        (APPROVAL_PARALLEL, 'Parallel Approval'),
    ]

    title = models.CharField(max_length=255)
    description = models.TextField(blank=True, null=True)
    status = models.CharField(
        max_length=20,
        choices=STATUS_CHOICES,
        default=STATUS_PENDING,
    )
    approval_type = models.CharField(
        max_length=20,
        choices=APPROVAL_CHOICES,
        default=APPROVAL_SERIAL,
    )

    def __str__(self):
        return self.title

    def can_transition(self, new_status):
        """ Check if the status transition is valid """
        valid_transitions = {
            self.STATUS_PENDING: [self.STATUS_IN_PROGRESS, self.STATUS_CANCELLED],
            self.STATUS_IN_PROGRESS: [self.STATUS_COMPLETED, self.STATUS_CANCELLED, self.STATUS_REJECTED, self.STATUS_APPROVED],
            self.STATUS_COMPLETED: [self.STATUS_WITHDRAWN],
            self.STATUS_CANCELLED: [],
            self.STATUS_REJECTED: [self.STATUS_PENDING],
            self.STATUS_APPROVED: [self.STATUS_COMPLETED],
            self.STATUS_WITHDRAWN: [self.STATUS_IN_PROGRESS],
        }
        return new_status in valid_transitions[self.status]

    def transition(self, new_status):
        """ Transition to a new status if valid """
        if self.can_transition(new_status):
            self.status = new_status
            self.save()
        else:
            raise ValueError(f"Invalid transition from {self.status} to {new_status}")
```

### 2. 创建序列化器

在 `tasks/serializers.py` 中更新序列化器以处理新字段：

```python
from rest_framework import serializers
from .models import Task

class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = '__all__'
```

### 3. 编写视图

在 `tasks/views.py` 中更新视图以处理复杂的状态转换逻辑：

```python
from rest_framework import viewsets, status
from rest_framework.response import Response
from rest_framework.decorators import action
from .models import Task
from .serializers import TaskSerializer

class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer

    @action(detail=True, methods=['post'])
    def transition(self, request, pk=None):
        task = self.get_object()
        new_status = request.data.get('status')
        if not new_status:
            return Response({"detail": "New status is required"}, status=status.HTTP_400_BAD_REQUEST)
        try:
            task.transition(new_status)
            return Response(TaskSerializer(task).data)
        except ValueError as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

### 4. 更新路由

在 `tasks/urls.py` 中更新路由：

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import TaskViewSet

router = DefaultRouter()
router.register(r'tasks', TaskViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

### 5. 迁移和运行服务器

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py runserver
```

这样，我们就实现了一个支持多种状态和转换的复杂流程状态机。现在，你可以通过 Django REST Framework 的 API 界面来管理任务，并通过 POST 请求 `/api/tasks/{id}/transition/` 来改变任务的状态。

如果还需要支持并行审批和串行审批的复杂逻辑，可以在 `Task` 模型中进一步扩展，增加关联的审批者和审批顺序等字段，并在 `transition` 方法中实现相应的逻辑。

------

当然可以，我们可以通过一个函数来实现批量更新和添加多个数据表的数据。这个函数会接收所有需要操作的数据，通过逻辑判断哪些需要更新，哪些需要添加，然后分别调用 `bulk_update` 和 `bulk_create` 来执行批量操作。

下面是实现这个功能的示例：

1. **定义模型**：

假设我们有两个模型 `Task` 和 `SubTask`：

```python
from django.db import models

class Task(models.Model):
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True, null=True)
    status = models.CharField(max_length=20, default='pending')

    def __str__(self):
        return self.title

class SubTask(models.Model):
    task = models.ForeignKey(Task, related_name='subtasks', on_delete=models.CASCADE)
    title = models.CharField(max_length=255)
    status = models.CharField(max_length=20, default='pending')

    def __str__(self):
        return self.title
```

2. **编写视图**：

在视图中使用一个函数来处理批量更新和添加操作。

```python
from django.shortcuts import get_object_or_404
from rest_framework import viewsets, status
from rest_framework.response import Response
from rest_framework.decorators import action
from django.db import transaction
from .models import Task, SubTask
from .serializers import TaskSerializer, SubTaskSerializer

def bulk_upsert(data, model, update_fields):
    """ 
    批量更新和添加数据的通用函数 
    data: 需要操作的数据列表
    model: 需要操作的模型
    update_fields: 需要更新的字段列表
    """
    instances_to_update = []
    instances_to_create = []

    for item_data in data:
        item_id = item_data.get('id')
        if item_id:
            try:
                instance = model.objects.get(id=item_id)
                for attr, value in item_data.items():
                    setattr(instance, attr, value)
                instances_to_update.append(instance)
            except model.DoesNotExist:
                instances_to_create.append(model(**item_data))
        else:
            instances_to_create.append(model(**item_data))

    with transaction.atomic():
        if instances_to_update:
            model.objects.bulk_update(instances_to_update, update_fields)
        if instances_to_create:
            model.objects.bulk_create(instances_to_create)

class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer

    @action(detail=False, methods=['post'])
    def bulk_upsert(self, request):
        tasks_data = request.data.get('tasks', [])
        if not tasks_data:
            return Response({"detail": "No tasks data provided"}, status=status.HTTP_400_BAD_REQUEST)

        # 分离 Task 和 SubTask 数据
        tasks = []
        subtasks = []
        
        for task_data in tasks_data:
            subtasks_data = task_data.pop('subtasks', [])
            tasks.append(task_data)
            for subtask_data in subtasks_data:
                subtask_data['task_id'] = task_data.get('id')
                subtasks.append(subtask_data)

        # 批量操作 Task 和 SubTask
        bulk_upsert(tasks, Task, ['title', 'description', 'status'])
        bulk_upsert(subtasks, SubTask, ['title', 'status', 'task_id'])

        return Response({"detail": "Tasks and subtasks have been upserted successfully"}, status=status.HTTP_200_OK)
```

3. **更新路由**：

在 `tasks/urls.py` 中添加 `bulk_upsert` 动作的路由。

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import TaskViewSet

router = DefaultRouter()
router.register(r'tasks', TaskViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

4. **迁移和运行服务器**：

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py runserver
```

**详细解释**：

- **模型**：
  - `Task` 和 `SubTask` 模型定义了任务和子任务的基本结构。
  - `SubTask` 通过外键与 `Task` 关联。

- **通用函数 `bulk_upsert`**：
  - 该函数接收数据列表、模型和需要更新的字段列表。
  - 遍历数据列表，如果提供了 `id`，则尝试更新对应的记录，否则创建新的记录。
  - 使用 `bulk_update` 和 `bulk_create` 批量更新和创建记录。
  - 在事务中执行批量操作，以确保数据一致性。

- **视图**：
  - `TaskViewSet` 中定义了 `bulk_upsert` 动作来处理任务和子任务的批量更新和添加。
  - 解析请求数据，分离 `Task` 和 `SubTask` 的数据。
  - 调用通用函数 `bulk_upsert` 批量操作任务和子任务。

- **路由**：
  - 使用 `DefaultRouter` 自动生成任务的路由。
  - 注册 `TaskViewSet`，将其关联到 `tasks` 路径，并包含默认路由。

通过这种方式，你可以使用一个通用函数高效地批量更新和添加多个数据表的数据。这种方法使代码更加简洁和复用。
