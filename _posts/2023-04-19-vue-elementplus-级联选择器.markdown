---
layout:     post
title:      "element级联选择器一级单选、二级多选"
subtitle:   "项目问题记录"
date:       2023-04-19
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - 项目问题记录
    - element-plus
    - vue
---

###### element级联选择器一级单选、二级多选

代码：

```js
const tag = ref(-1)
function change(item){
    item.forEach(v => {
        this.tag = v[0]
    })
    let filterd = item.filter(v => v[0] == this.tag)
    this.selectDepartment = filterd
}
```

在级联选择器中添加change事件
