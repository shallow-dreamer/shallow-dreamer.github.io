---
layout:     post
title:      "vue-django图片与数据传递"
subtitle:   "后端使用Django将图片以base64格式以及数据传递到vue前端展示"
date:       2023-04-30
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - vue
    - django
---

代码：

```vue
<template>
  <div class="item">
    <div>{{ message }}</div>
    <img :src="img" alt="" style="width: 300px; height: 300px;" />
    <button @click="getPic">获取图片</button>
  </div>
</template>

<script>
import axios from 'axios';
import { reactive, toRefs } from 'vue'

export default {
  setup() {
    const data = reactive({
      img: '',
      message: '',
      getPic
    })
    function getPic() {
      axios.get('/api/test', {

      })
        .then(response => {
          // 处理响应数据
          data.img = 'data:image/png;base64,' + response.data.img
          data.message = response.data / message
        })
        .catch(error => {
          // 处理错误
        })
    }
    return {
      ...toRefs(data)
    }
  }
}
</script>

<style scoped>
</style>

```

```python
def test(request) -> JsonResponse:
    try:
        img = open("D:\Program Files\JetBrains\PyCharm Community Edition 2022.3.1\skin\丽娘.jpg", "rb")
        file_data = img.read()
        encode_file_data = base64.b64encode(file_data).decode("utf-8")
        response_data = {"img": encode_file_data, "message": "hello"}
        return JsonResponse(response_data, content_type="application/json")
    except Exception as ex:
        return JsonResponse({"errStr": str(ex)})
```

