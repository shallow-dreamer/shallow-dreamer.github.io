---
layout:     post
title:      "eltable触底加载demo"
subtitle:   ""
date:       2023-08-15
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - vue
    - eltable
---

在Vue 3 中使用 Element Plus 的 `el-table` 组件实现触底加载类似于 Vue 2，但需要稍微适应 Composition API 的使用方式。以下是一个示例，展示如何在 Vue 3 中实现触底加载：

```vue
<template>
  <div>
    <el-table :data="tableData" height="400px" border>
      <el-table-column prop="name" label="Name"></el-table-column>
      <!-- Other columns go here -->
    </el-table>
    <div v-if="isLoading" class="loading-indicator">Loading...</div>
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount } from 'vue';

export default {
  setup() {
    const tableData = ref([]);
    const isLoading = ref(false);
    const currentPage = ref(1);
    const pageSize = 20;
    const totalItems = ref(0);

    const loadData = () => {
      // Simulate an API call to load data
      isLoading.value = true;
      setTimeout(() => {
        // Assuming newData is the loaded data from API
        const newData = [...];
        tableData.value = [...tableData.value, ...newData];
        totalItems.value = tableData.value.length;
        isLoading.value = false;
        currentPage.value++;
      }, 1000);
    };

    const handleScroll = (event) => {
      const target = event.target;
      const threshold = 100; // The threshold before triggering the load (in pixels)

      // Check if the user has scrolled to the bottom of the table
      if (target.scrollTop + target.clientHeight >= target.scrollHeight - threshold) {
        loadData();
      }
    };

    onMounted(() => {
      // Attach the scroll event listener to the table container
      document.querySelector(".el-table__body-wrapper").addEventListener("scroll", handleScroll);
    });

    onBeforeUnmount(() => {
      // Detach the scroll event listener when the component is unmounted
      document.querySelector(".el-table__body-wrapper").removeEventListener("scroll", handleScroll);
    });

    return {
      tableData,
      isLoading,
      loadData,
      handleScroll
    };
  }
};
</script>

<style>
.loading-indicator {
  text-align: center;
  padding: 10px;
}
</style>
```

在这个 Vue 3 的示例中，我们使用了 Composition API 提供的 `ref` 来创建响应式数据，以及 `onMounted` 和 `onBeforeUnmount` 钩子来处理事件监听和销毁。其余逻辑与 Vue 2 的示例类似。

记得根据你的项目和 API 结构进行适当的调整，以满足你的实际需求。

在Vue 3 中使用 Element Plus 的 `el-table` 组件实现触底加载（无限滚动加载）的过程与之前介绍的类似，但是需要在 Vue 3 的语法下进行适当的修改。以下是一个在 Vue 3 中实现触底加载的示例：

首先，确保你已经安装了 Element Plus 和 Vue 3 的依赖。然后按照以下步骤进行实现：

1. **创建组件：** 创建一个 Vue 组件用于展示触底加载的表格。

```vue
<template>
  <div>
    <el-table :data="tableData" height="400px" border>
      <el-table-column prop="name" label="Name"></el-table-column>
      <!-- 其他列 -->
    </el-table>
    <div v-if="isLoading" class="loading-indicator">Loading...</div>
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount } from 'vue';

export default {
  setup() {
    const tableData = ref([]);
    const isLoading = ref(false);
    const currentPage = ref(1);
    const pageSize = 20;
    const totalItems = ref(0);

    const loadData = () => {
      // 模拟 API 调用来加载数据
      isLoading.value = true;
      setTimeout(() => {
        // 假设 newData 是从 API 加载的数据
        const newData = [...];
        tableData.value = [...tableData.value, ...newData];
        totalItems.value = tableData.value.length;
        isLoading.value = false;
        currentPage.value++;
      }, 1000);
    };

    const handleScroll = (event) => {
      const target = event.target;
      const threshold = 100; // 在触发加载前的阈值（像素）
      
      // 检查用户是否滚动到了表格底部
      if (target.scrollTop + target.clientHeight >= target.scrollHeight - threshold) {
        loadData();
      }
    };

    onMounted(() => {
      // 在组件挂载时添加滚动事件监听器
      const tableWrapper = document.querySelector('.el-table__body-wrapper');
      tableWrapper.addEventListener('scroll', handleScroll);
    });

    onBeforeUnmount(() => {
      // 在组件销毁前移除滚动事件监听器
      const tableWrapper = document.querySelector('.el-table__body-wrapper');
      tableWrapper.removeEventListener('scroll', handleScroll);
    });

    return {
      tableData,
      isLoading,
    };
  },
};
</script>

<style>
.loading-indicator {
  text-align: center;
  padding: 10px;
}
</style>
```

在这个示例中，我们使用 Vue 3 的 Composition API（`setup` 函数）来管理组件的状态和逻辑。`ref` 函数用于创建响应式数据，`onMounted` 和 `onBeforeUnmount` 钩子用于在组件挂载和销毁时添加/移除滚动事件监听器。

确保将上述代码整理为一个完整的 Vue 组件，然后在你的应用中使用这个组件来实现触底加载的表格。在实际项目中，你需要根据你的业务逻辑和后端接口进行适当的调整和处理。

在Vue 3 中使用 Element Plus 的 `el-table` 组件实现触底加载的原理与之前的示例类似，但是要注意在 Vue 3 中的一些语法和使用方式上的变化。以下是在 Vue 3 中使用 `el-table` 实现触底加载的简单示例：

首先，确保你已经安装了 Vue 3 和 Element Plus。

```bash
npm install vue@next
npm install element-plus
```

然后，你可以创建一个组件来实现触底加载：

```vue
<template>
  <div>
    <el-table :data="tableData" height="400px" border>
      <el-table-column prop="name" label="Name"></el-table-column>
      <!-- Other columns go here -->
    </el-table>
    <div v-if="isLoading" class="loading-indicator">Loading...</div>
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount } from 'vue';

export default {
  setup() {
    const tableData = ref([]);
    const isLoading = ref(false);
    const currentPage = ref(1);
    const pageSize = 20;
    const totalItems = ref(0);

    // Simulate an API call to load data
    const loadData = () => {
      isLoading.value = true;
      setTimeout(() => {
        // Assuming newData is the loaded data from API
        const newData = [...];
        tableData.value = [...tableData.value, ...newData];
        totalItems.value = tableData.value.length;
        isLoading.value = false;
        currentPage.value++;
      }, 1000);
    };

    // Handle scroll event
    const handleScroll = (event) => {
      const target = event.target;
      const threshold = 100; // The threshold before triggering the load (in pixels)

      // Check if the user has scrolled to the bottom of the table
      if (target.scrollTop + target.clientHeight >= target.scrollHeight - threshold) {
        loadData();
      }
    };

    onMounted(() => {
      // Attach the scroll event listener to the table container
      const tableBodyWrapper = document.querySelector(".el-table__body-wrapper");
      tableBodyWrapper.addEventListener("scroll", handleScroll);

      // Load initial data
      loadData();
    });

    onBeforeUnmount(() => {
      // Detach the scroll event listener when the component is unmounted
      const tableBodyWrapper = document.querySelector(".el-table__body-wrapper");
      tableBodyWrapper.removeEventListener("scroll", handleScroll);
    });

    return {
      tableData,
      isLoading
    };
  }
};
</script>

<style>
.loading-indicator {
  text-align: center;
  padding: 10px;
}
</style>
```

在这个 Vue 3 的示例中，我们使用了 Composition API 的 `ref` 和生命周期钩子函数 `onMounted` 和 `onBeforeUnmount` 来实现组件逻辑。其余部分的逻辑和之前的示例相似。

注意，你可能需要根据自己的需求进行调整，例如根据后端接口获取数据等。另外，建议使用 Element Plus 的 `el-pagination` 组件来管理分页，以便更好地展示和控制数据的分页加载。

在Vue 3 中使用 Element Plus 的 `el-table` 组件实现触底加载（无限滚动加载）的方法与上述类似。以下是一个 Vue 3 的示例，展示如何在 `el-table` 中实现触底加载：

```vue
<template>
  <div>
    <el-table :data="tableData" height="400px" border>
      <el-table-column prop="name" label="Name"></el-table-column>
      <!-- Other columns go here -->
    </el-table>
    <div v-if="isLoading" class="loading-indicator">Loading...</div>
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount } from 'vue';

export default {
  setup() {
    const tableData = ref([]); // The current displayed data
    const isLoading = ref(false); // Loading indicator flag
    const currentPage = ref(1); // Current page of data
    const pageSize = 20; // Number of items per page
    const totalItems = ref(0); // Total number of items

    const loadData = () => {
      // Simulate an API call to load data
      isLoading.value = true;
      setTimeout(() => {
        // Assuming newData is the loaded data from API
        const newData = [...];
        tableData.value = [...tableData.value, ...newData];
        totalItems.value = tableData.value.length;
        isLoading.value = false;
        currentPage.value++;
      }, 1000);
    };

    const handleScroll = (event) => {
      const target = event.target;
      const threshold = 100; // The threshold before triggering the load (in pixels)
      
      // Check if the user has scrolled to the bottom of the table
      if (target.scrollTop + target.clientHeight >= target.scrollHeight - threshold) {
        loadData();
      }
    };

    onMounted(() => {
      // Attach the scroll event listener to the table container
      const tableWrapper = document.querySelector(".el-table__body-wrapper");
      tableWrapper.addEventListener("scroll", handleScroll);
    });

    onBeforeUnmount(() => {
      // Detach the scroll event listener when the component is unmounted
      const tableWrapper = document.querySelector(".el-table__body-wrapper");
      tableWrapper.removeEventListener("scroll", handleScroll);
    });

    return {
      tableData,
      isLoading
    };
  }
};
</script>

<style>
.loading-indicator {
  text-align: center;
  padding: 10px;
}
</style>
```

这个示例中，我使用了 Vue 3 的 Composition API，其中 `ref` 是用来创建响应式数据的函数，而 `onMounted` 和 `onBeforeUnmount` 是用来处理组件的生命周期钩子。其余的逻辑与之前的示例类似，可以参考来实现在 Vue 3 中使用 Element Plus 的 `el-table` 组件实现触底加载。

在Vue 3 中，使用 Element Plus 的 `el-table` 组件实现触底加载与 Vue 2 的方法类似，但是由于 Vue 3 的语法略有不同，需要做一些调整。以下是在 Vue 3 中使用 `el-table` 实现触底加载的示例：

```vue
<template>
  <div>
    <el-table :data="tableData" height="400px" border>
      <el-table-column prop="name" label="Name"></el-table-column>
      <!-- Other columns go here -->
    </el-table>
    <div v-if="isLoading" class="loading-indicator">Loading...</div>
  </div>
</template>

<script>
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue';

export default {
  setup() {
    const tableData = ref([]);       // The current displayed data
    const isLoading = ref(false);    // Loading indicator flag
    const currentPage = ref(1);      // Current page of data
    const pageSize = 20;             // Number of items per page
    const totalItems = ref(0);       // Total number of items

    onMounted(() => {
      // Attach the scroll event listener to the table container
      const tableContainer = document.querySelector(".el-table__body-wrapper");
      tableContainer.addEventListener("scroll", handleScroll);
      
      // Load initial data
      loadData();
    });

    onBeforeUnmount(() => {
      // Detach the scroll event listener when the component is unmounted
      const tableContainer = document.querySelector(".el-table__body-wrapper");
      tableContainer.removeEventListener("scroll", handleScroll);
    });

    const loadData = () => {
      // Simulate an API call to load data
      isLoading.value = true;
      setTimeout(() => {
        // Assuming newData is the loaded data from API
        const newData = [...];
        tableData.value = [...tableData.value, ...newData];
        totalItems.value = tableData.value.length;
        isLoading.value = false;
        currentPage.value++;
      }, 1000);
    };

    const handleScroll = (event) => {
      const target = event.target;
      const threshold = 100; // The threshold before triggering the load (in pixels)
      
      // Check if the user has scrolled to the bottom of the table
      if (target.scrollTop + target.clientHeight >= target.scrollHeight - threshold) {
        loadData();
      }
    };

    return {
      tableData,
      isLoading,
      loadData,
      handleScroll
    };
  }
};
</script>

<style>
.loading-indicator {
  text-align: center;
  padding: 10px;
}
</style>
```

在这个示例中，我们使用 Vue 3 的 Composition API，通过 `ref` 和 `reactive` 来创建响应式的数据，然后使用 `onMounted` 和 `onBeforeUnmount` 钩子来处理事件监听的添加和移除。逻辑与 Vue 2 示例类似，只是语法上略有不同。

请确保在使用此示例时，将实际的 API 调用和数据结构进行相应的修改以适应你的项目需求。
