---
layout:     post
title:      "element时间选择器：日期截止当前、中文、右侧为当前月"
subtitle:   "项目问题记录"
date:       2023-04-19
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - 项目问题记录
    - element-plus
    - vue
---

###### element时间选择器

```vue
<el-config-provider :locale="locale">
	<el-date-picker v-model="selectDate" type="daterange" range-separator="-" start-placeholder="Start date" end-placeholder="End date" :disabled-date="disableDate" :default-value="Date.now() - 30 * 24 * 3600 * 1000" value-format="YYYY-MM-DD" unlink-panels />
</el-config-provider>
```

日期截止当前

```js
function disableDate(){
    return Date.now() < time.getTime()
}
```

中文

```js
import { zhCn } from 'element-plus/es/locale'
return { locale: zhCn }
```

右侧当前月

```js
:default-value="Date.now() - 30 * 24 * 3600 * 1000"
```

