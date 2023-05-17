---
layout:     post
title:      "vue-django活跃度"
subtitle:   "活跃度"
date:       2023-05-17
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - vue
    - django
---

以下是一个使用Vue和Python Flask框架来实现从数据库获取每天活跃度数据，然后在前端用Echarts以日期的方式展示折线图的示例代码：

## 后端代码

```python
# app.py

from flask import Flask, jsonify
from flask_cors import CORS
import datetime
import random

app = Flask(__name__)
CORS(app)

@app.route('/api/active/<string:period>')
def get_active(period):
    today = datetime.date.today()
    data = []
    if period == 'daily':
        for i in range(7):
            date = today - datetime.timedelta(days=i)
            active_users = random.randint(100, 500)
            data.append({'date': str(date), 'active_users': active_users})
    elif period == 'weekly':
        for i in range(4):
            start_date = today - datetime.timedelta(days=7*(i+1))
            end_date = today - datetime.timedelta(days=7*i)
            active_users = random.randint(1000, 5000)
            data.append({'date': '{} ~ {}'.format(str(start_date), str(end_date)), 'active_users': active_users})
    elif period == 'monthly':
        for i in range(6):
            year, month = divmod(today.month - i, 12)
            if month == 0:
                month = 12
                year -= 1
            start_date = datetime.date(year, month, 1)
            end_date = datetime.date(year, month, 28) + datetime.timedelta(days=3)
            end_date = end_date - datetime.timedelta(days=end_date.day-1)
            active_users = random.randint(10000, 50000)
            data.append({'date': '{} ~ {}'.format(str(start_date), str(end_date)), 'active_users': active_users})
    else:
        return jsonify({'error': 'Invalid period'}), 400
    return jsonify({'data': data})
```

## 前端代码

```html
<!-- Active.vue -->

<template>
  <div class="active-chart"></div>
</template>

<script>
import * as echarts from 'echarts';
import axios from 'axios';

export default {
  name: 'Active',
  mounted() {
    this.renderChart();
  },
  methods: {
    async renderChart() {
      const response = await axios.get('http://localhost:5000/api/active/daily');
      const data = response.data.data;
      
      const chart = echarts.init(document.querySelector('.active-chart'));
      const option = {
        title: {
          text: 'Daily Active Users',
        },
        tooltip: {
          trigger: 'axis',
          formatter: '{b} : {c}',
        },
        xAxis: {
          type: 'category',
          data: data.map((item) => item.date),
        },
        yAxis: {
          type: 'value',
        },
        series: [
          {
            type: 'line',
            data: data.map((item) => item.active_users),
          },
        ],
      };
      chart.setOption(option);
    },
  },
};
</script>

<style scoped>
.active-chart {
  width: 100%;
  height: 400px;
}
</style>
```

以上示例代码展示了如何使用Vue和Python Flask框架实现从数据库获取每天活跃度数据，然后在前端用Echarts以日期的方式展示折线图。在后端代码中，我们使用了Flask框架来创建API接口，并通过随机数模拟了一些活跃度数据。在前端代码中，我们使用了axios库来从API接口获取数据，并使用Echarts库来绘制折线图。
