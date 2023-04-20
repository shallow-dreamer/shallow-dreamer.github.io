---
layout:     post
title:      "利用js创建a标签下载文件"
subtitle:   "使用js"
date:       2023-04-20
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - js
---

代码：

```javascript
let url = window.URL.createObjectURL(new Blob([文件流（一般为res.data)], {type: "Blob类型"})
let link = document.createElement('a')
link.style.dispaly = 'none'
link.href = url
link.setAttribute('download', '下载的文件名')
document.body.appendChild(link)
link.click()
document.body.removeChild(link)
```
