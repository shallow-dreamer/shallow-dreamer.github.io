---
layout:     post
title:      "在ElDrawer中使用echarts"
subtitle:   ""
date:       2023-07-17
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - vue3
    - elementPlus
    - echarts
---

在 Vue 3 中使用 Element Plus，你可以使用 `v-show` 指令结合 CSS 样式来模拟 el-drawer 的 DOM 元素始终存在的效果。Element Plus 的 el-drawer 组件是基于 Vue 3 的动态组件，它的 DOM 元素在组件的显示和隐藏之间是动态创建和销毁的。

以下是一个示例代码，演示如何让 el-drawer 的 DOM 元素始终存在：

```vue
<template>
  <div>
    <div class="persistent-drawer" v-show="drawerVisible"></div>
    <el-drawer v-model="drawerVisible" @open="onDrawerOpen" @close="onDrawerClose">
      <div id="chartContainer" style="width: 100%; height: 300px;"></div>
    </el-drawer>
    <el-button @click="openDrawer">打开 Drawer</el-button>
  </div>
</template>

<script>
import { ref } from 'vue';
import * as echarts from 'echarts';

export default {
  setup() {
    const drawerVisible = ref(false);
    let chartInstance = null;

    const openDrawer = () => {
      drawerVisible.value = true;
    };

    const onDrawerOpen = () => {
      // 初始化 echarts 实例
      chartInstance = echarts.init(document.getElementById('chartContainer'));
      // 进行图表的配置和数据加载等操作
      chartInstance.setOption({
        // 配置项
        // ...
      });
    };

    const onDrawerClose = () => {
      // 销毁 echarts 实例
      if (chartInstance) {
        chartInstance.dispose();
        chartInstance = null;
      }
    };

    return {
      drawerVisible,
      openDrawer,
      onDrawerOpen,
      onDrawerClose
    };
  }
};
</script>

<style scoped>
.persistent-drawer {
  width: 0;
  height: 0;
  visibility: hidden;
  pointer-events: none;
}
</style>
```

在这个示例中，我们使用了一个名为 `.persistent-drawer` 的样式类来模拟持久存在的 DOM 元素。通过 `v-show` 指令，在 el-drawer 组件的前面添加了一个具有 `.persistent-drawer` 类的空元素。这个元素不会显示出来，但它的存在可以确保 el-drawer 的 DOM 结构始终存在于页面中。

同时，我们使用 Vue 3 的 `ref` 函数创建了 `drawerVisible` 响应式数据，用于控制 el-drawer 的显示和隐藏。在 `openDrawer` 方法中，将 `drawerVisible` 的值设置为 true，从而打开 el-drawer。

在 el-drawer 组件的 `@open` 事件回调函数中初始化 echarts 实例，并在 `@close` 事件回调函数中销毁 echarts 实例。

通过这种方式，el-drawer 的 DOM 元素会持久存在，但只有在 `drawerVisible` 的值为 true 时才会显示出来。
