---
layout:     post
title:      "上传xlsx文件内容到数据库"
subtitle:   ""
date:       2023-06-09
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - python
    - mysql
---

当然！下面是一个使用Vue 3、Element Plus、Django 和 MySQL 的示例代码，可以实现上传 XLSX 文件并将其内容导入到数据库中。 

前端代码（Vue 3 + Element Plus）：

```html
<template>
  <div>
    <el-upload
      class="upload-demo"
      drag
      action="/api/upload"
      :on-success="handleUploadSuccess"
    >
      <i class="el-icon-upload"></i>
      <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
    </el-upload>
  </div>
</template>

<script>
import { ref } from 'vue';

export default {
  setup() {
    const handleUploadSuccess = (response) => {
      // 处理上传成功的回调
      console.log(response.data); // 在控制台打印返回的数据
    };

    return {
      handleUploadSuccess,
    };
  },
};
</script>
```

后端代码（Django）：

```python
# views.py
from django.views import View
from django.http import JsonResponse

class FileUploadView(View):
    def post(self, request):
        file = request.FILES.get('file')
        if file:
            # 处理上传的文件，将内容导入到数据库中
            # 这里假设你已经安装了 openpyxl 库用于处理 Excel 文件
            from openpyxl import load_workbook
            
            workbook = load_workbook(file)
            sheet = workbook.active
            data = []
            
            for row in sheet.iter_rows(values_only=True):
                # 假设表格有三列，分别是 name、age 和 email
                name, age, email = row
                data.append({
                    'name': name,
                    'age': age,
                    'email': email,
                })
            
            # 将数据存入数据库
            # 这里假设你已经设置好了数据库连接，并创建了对应的模型
            from .models import YourModel
            
            for item in data:
                YourModel.objects.create(
                    name=item['name'],
                    age=item['age'],
                    email=item['email'],
                )
            
            return JsonResponse({'data': '上传成功'})
        
        return JsonResponse({'error': '未找到上传的文件'})
```

```python
# urls.py
from django.urls import path
from .views import FileUploadView

urlpatterns = [
    path('upload', FileUploadView.as_view(), name='file-upload'),
]
```

请确保已经正确安装并配置了 Vue 3、Element Plus、Django、MySQL 和 openpyxl 库。将前端代码和后端代码分别保存为 `YourComponent.vue` 和 `views.py`，并将 `views.py` 中的路由添加到你的 Django 项目的 `urls.py` 文件中。

这是一个简单的示例，仅供参考。在实际开发中，你可能需要根据自己的需求进行适当的修改和扩展。

当然！我可以帮助您编写一个使用Vue 3、Element Plus、Django和MySQL的示例，以上传XLSX文件并将其内容导入到数据库中。

首先，您需要设置一个Django项目并配置MySQL数据库。确保您已经安装了Django和MySQL，并配置了适当的数据库连接。

接下来，我们将创建一个Django视图来处理文件上传和数据导入的逻辑。假设您的Django应用程序名为`myapp`，请按照以下步骤进行操作：

1. 在`myapp/views.py`文件中添加以下代码：

```python
import pandas as pd
from django.shortcuts import render
from django.contrib import messages

def upload_file(request):
    if request.method == 'POST' and request.FILES['file']:
        file = request.FILES['file']
        if file.name.endswith('.xlsx'):
            try:
                df = pd.read_excel(file)
                # 在这里执行将数据导入到数据库的逻辑
                # 例如，将数据保存到MySQL数据库中的名为`mytable`的表中
                df.to_sql('mytable', your_mysql_connection, if_exists='replace', index=False)
                messages.success(request, '文件上传和数据导入成功！')
            except Exception as e:
                messages.error(request, f'发生错误：{str(e)}')
        else:
            messages.error(request, '请选择一个有效的XLSX文件！')
    return render(request, 'upload.html')
```

2. 在`myapp/urls.py`文件中添加以下代码：

```python
from django.urls import path
from myapp.views import upload_file

urlpatterns = [
    path('upload/', upload_file, name='upload'),
]
```

3. 创建一个名为`upload.html`的模板文件，并放置在`myapp/templates/`目录下。在该文件中，您可以使用Vue 3和Element Plus来构建文件上传界面。以下是一个简单的示例：

```html
<template>
  <div>
    <el-upload
      class="upload-demo"
      action="/upload/"
      :on-success="handleSuccess"
      :on-error="handleError"
      :before-upload="beforeUpload"
      :show-file-list="false"
    >
      <el-button slot="trigger" size="small" type="primary">选择文件</el-button>
      <div slot="tip" class="el-upload__tip">只能上传XLSX文件</div>
    </el-upload>
  </div>
</template>

<script>
export default {
  methods: {
    handleSuccess(response) {
      this.$message.success('文件上传成功！');
    },
    handleError(err) {
      this.$message.error('文件上传失败！');
    },
    beforeUpload(file) {
      const isXLSX = file.type === 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet';
      if (!isXLSX) {
        this.$message.error('请选择一个有效的XLSX文件！');
      }
      return isXLSX;
    },
  },
};
</script>

<style scoped>
.upload-demo {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 200px;
  border: 1px dashed #d9d9d9;
  border-radius: 6px;
  background-color: #f5f7fa;
}
</style>
```

要使用Django将XLSX文件中的内容导入到数据库中，你需要使用适当的库来解析和读取XLSX文件，例如`openpyxl`。下面是一个示例，演示了如何使用`openpyxl`和Django来实现这个功能：

1. 首先，确保你已经安装了`openpyxl`库：

```bash
pip install openpyxl
```

2. 在Django应用的视图函数中，处理上传的XLSX文件并将其内容导入到数据库中。在`views.py`文件中，添加以下代码：

```python
import openpyxl
from django.shortcuts import render
from .models import UploadedData

def upload_file(request):
    if request.method == 'POST' and request.FILES['file']:
        file = request.FILES['file']
        # 读取上传的XLSX文件
        workbook = openpyxl.load_workbook(file)
        sheet = workbook.active

        # 遍历每一行数据并保存到数据库
        for row in sheet.iter_rows(values_only=True):
            column1 = row[0]
            column2 = row[1]
            # 创建数据库对象并保存
            data = UploadedData(column1=column1, column2=column2)
            data.save()

        return render(request, 'success.html')

    return render(request, 'upload.html')
```

3. 创建一个上传文件的HTML模板。在`templates`文件夹下创建一个名为`upload.html`的模板文件，并添加以下代码：

```html
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  <input type="file" name="file" accept=".xlsx">
  <button type="submit">上传</button>
</form>
```

4. 创建一个成功页面的HTML模板。在`templates`文件夹下创建一个名为`success.html`的模板文件，并添加以下代码：

```html
<h2>上传成功！</h2>
```

5. 在`urls.py`文件中添加相应的URL映射，将视图函数与URL关联起来：

```python
from django.urls import path
from . import views

urlpatterns = [
    path('upload/', views.upload_file, name='upload'),
]
```

完成以上步骤后，你可以启动Django开发服务器，然后在浏览器中访问`http://localhost:8000/upload/`，上传XLSX文件，并将文件内容导入到数据库中。

请注意，上述示例仅涵盖了基本的文件上传和数据导入逻辑。在实际应用中，你可能需要进行更多的数据验证和错误处理，并根据实际需求对模型、视图和模板进行调整。
