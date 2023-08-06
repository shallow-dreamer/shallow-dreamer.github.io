---
layout:     post
title:      "将xlsx文件导入到el-table表格"
subtitle:   ""
date:       2023-08-06
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - vue
    - xlsx
    - el-table
---

要将XLSX文件导入到前端Element Plus的ElTable表格中，您可以使用`xlsx`库来读取XLSX文件，并将数据转换为表格所需的数据格式，然后将其展示在ElTable中。以下是一个基本的实现步骤：

1. 安装`xlsx`库：
首先，您需要安装`xlsx`库来处理XLSX文件。使用npm或yarn进行安装。

使用npm：
```bash
npm install xlsx --save
```

使用yarn：
```bash
yarn add xlsx
```

2. 导入XLSX文件：
在前端中，使用文件选择器（input type="file"）来让用户选择要导入的XLSX文件。通过监听文件选择器的change事件，获取用户选择的文件。

3. 读取XLSX文件数据：
使用`xlsx`库的`readFile`函数来读取XLSX文件并获取数据。将数据转换为表格所需的数据格式，例如二维数组或包含对象的数组。

4. 展示数据在ElTable中：
将读取到的数据展示在ElTable中，通过绑定表格的data属性即可。

下面是一个简单的示例代码：

```vue
<template>
  <div>
    <input type="file" @change="handleFileChange" accept=".xlsx">
    <el-table :data="tableData" v-if="tableData.length > 0">
      <el-table-column prop="name" label="姓名"></el-table-column>
      <el-table-column prop="age" label="年龄"></el-table-column>
      <!-- 更多表格列定义 -->
    </el-table>
  </div>
</template>

<script>
import XLSX from 'xlsx';

export default {
  data() {
    return {
      tableData: []
    };
  },
  methods: {
    handleFileChange(event) {
      const file = event.target.files[0];
      const reader = new FileReader();

      reader.onload = (e) => {
        const data = new Uint8Array(e.target.result);
        const workbook = XLSX.read(data, { type: 'array' });
        const sheetName = workbook.SheetNames[0]; // 假设数据在第一个工作表
        const worksheet = workbook.Sheets[sheetName];
        const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 });

        // 适配数据到ElTable所需的格式
        const tableData = jsonData.map(row => {
          return {
            name: row[0], // 假设姓名在第一列
            age: row[1], // 假设年龄在第二列
            // 更多字段
          };
        });

        this.tableData = tableData;
      };

      reader.readAsArrayBuffer(file);
    }
  }
};
</script>
```

请注意，上述示例代码仅作为一个简单的示例，实际应用中可能需要根据XLSX文件的具体结构和表格的需求进行适当调整。例如，您可能需要处理表头行、数据类型转换和错误处理等情况。

要将xlsx文件导入到前端Element-Plus的el-table表格中，您可以采用以下步骤：

1. 安装依赖：

在Vue前端项目中，您需要使用`xlsx`库来处理xlsx文件的解析。使用npm或yarn安装`xlsx`库：

使用npm：
```bash
npm install xlsx --save
```

使用yarn：
```bash
yarn add xlsx
```

2. 创建文件选择组件：

首先，您需要创建一个文件选择组件，允许用户选择要导入的xlsx文件。

```vue
<template>
  <div>
    <input type="file" ref="fileInput" @change="handleFileChange" />
  </div>
</template>

<script>
export default {
  methods: {
    handleFileChange(event) {
      const file = event.target.files[0];
      if (file) {
        this.readFile(file);
      }
    },
    readFile(file) {
      const reader = new FileReader();
      reader.onload = (e) => {
        const data = new Uint8Array(e.target.result);
        this.parseXLSXData(data);
      };
      reader.readAsArrayBuffer(file);
    },
    parseXLSXData(data) {
      // 使用xlsx库解析数据并处理
      // 在这里可以将数据处理成el-table所需的格式
    },
  },
};
</script>
```

3. 解析xlsx文件数据：

在`parseXLSXData`方法中，您可以使用`xlsx`库来解析数据并处理它，使其符合el-table所需的格式。xlsx库提供了`XLSX.read`方法来读取xlsx文件数据。

```javascript
import XLSX from 'xlsx';

export default {
  // ...
  methods: {
    // ...
    parseXLSXData(data) {
      const workbook = XLSX.read(data, { type: 'array' });
      const firstSheetName = workbook.SheetNames[0];
      const worksheet = workbook.Sheets[firstSheetName];
      const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 });
      
      // jsonData现在包含了解析后的数据，根据需要转换成el-table所需的格式
      // 例如，如果jsonData是一个二维数组，可以转换为一个数组对象，每个对象代表一行数据
      const tableData = jsonData.map((row) => {
        const rowData = {};
        // 将每一行数据转换为对象，设置对应的列名
        // 例如：rowData = { column1: row[0], column2: row[1], ... }
        return rowData;
      });

      // 将转换后的数据设置到el-table中
      this.tableData = tableData;
    },
  },
};
```

在上述代码中，我们使用`XLSX.read`方法将xlsx文件数据读取为Workbook对象，然后获取第一个Sheet的数据并转换为json格式。接下来，我们根据需要处理jsonData，并将处理后的数据设置到el-table的数据源中（`this.tableData`）。

4. 在el-table中显示导入的数据：

现在，您已经成功将xlsx文件的数据导入到前端，并转换成了el-table所需的数据格式。您可以在el-table中显示导入的数据了：

```vue
<template>
  <div>
    <el-table :data="tableData" border>
      <!-- 根据需要添加el-table-column，指定对应的列名 -->
    </el-table>
  </div>
</template>

<script>
export default {
  data() {
    return {
      tableData: [], // 导入的数据将存储在这里
    };
  },
};
</script>
```

在上述代码中，我们使用了`el-table`和`el-table-column`来展示导入的数据。根据需要，您可以在`el-table`中添加`el-table-column`来指定对应的列名和样式。

通过以上步骤，您可以成功将xlsx文件导入到前端Element-Plus的el-table表格中，并在el-table中展示导入的数据。请注意，上述代码只是一个简单示例，您可能需要根据实际需求进行进一步的处理和样式调整。

要将xlsx文件导入到前端的Element Plus的el-table表格中，您需要进行以下步骤：

1. 安装依赖：

首先，您需要安装xlsx和file-saver库来处理xlsx文件的解析和保存。

使用npm：
```bash
npm install xlsx file-saver --save
```

使用yarn：
```bash
yarn add xlsx file-saver
```

2. 创建文件上传组件：

在Vue项目中创建一个文件上传组件，用于让用户选择并上传xlsx文件。

示例代码：

```vue
<template>
  <div>
    <input type="file" ref="fileInput" @change="handleFileChange">
    <el-button type="primary" @click="importData">导入表格</el-button>
    <el-table :data="tableData" v-if="tableData.length > 0">
      <!-- 表格列配置 -->
      <!-- 示例： -->
      <el-table-column prop="name" label="姓名"></el-table-column>
      <el-table-column prop="age" label="年龄"></el-table-column>
    </el-table>
  </div>
</template>

<script>
import { read, utils } from 'xlsx';
import { saveAs } from 'file-saver';

export default {
  data() {
    return {
      tableData: [] // 存储导入的数据
    };
  },
  methods: {
    handleFileChange() {
      const file = this.$refs.fileInput.files[0];
      this.readFileData(file);
    },
    readFileData(file) {
      const reader = new FileReader();
      reader.onload = (event) => {
        const data = event.target.result;
        const workbook = read(data, { type: 'binary' });
        const firstSheetName = workbook.SheetNames[0];
        const worksheet = workbook.Sheets[firstSheetName];
        const jsonData = utils.sheet_to_json(worksheet, { header: 1 });
        this.tableData = jsonData.slice(1).map(row => {
          return {
            name: row[0],
            age: row[1],
            // 根据实际列配置继续添加字段
          };
        });
      };
      reader.readAsBinaryString(file);
    },
    importData() {
      // 在这里可以将tableData导入到后端进行处理
      // 如果需要，也可以将tableData保存为xlsx文件并下载
      const workbook = utils.book_new();
      const worksheet = utils.json_to_sheet(this.tableData);
      utils.book_append_sheet(workbook, worksheet, 'Sheet1');
      const fileData = utils.write(workbook, { type: 'binary' });
      const blob = new Blob([s2ab(fileData)], { type: 'application/octet-stream' });
      saveAs(blob, 'exported_data.xlsx');
    }
  }
}

// 将字符串转换为ArrayBuffer
function s2ab(s) {
  const buf = new ArrayBuffer(s.length);
  const view = new Uint8Array(buf);
  for (let i = 0; i < s.length; i++) {
    view[i] = s.charCodeAt(i) & 0xFF;
  }
  return buf;
}
</script>
```

在上述示例代码中，我们创建了一个文件上传组件，用户可以选择xlsx文件并导入到el-table表格中。通过xlsx库解析文件内容，然后将数据存储在tableData中，再通过el-table展示数据。通过file-saver库，我们还提供了导出数据的功能，将tableData保存为xlsx文件并下载。

请注意，上述代码中的表格列配置示例可能需要根据您的xlsx文件的实际列配置进行调整。同时，对于大型xlsx文件，建议对解析数据的逻辑进行优化，以提高性能。

希望这可以帮助您将xlsx文件导入到前端的Element Plus的el-table表格中。如果需要更复杂的处理或其他功能，您可以根据项目的需求进行相应的扩展和调整。
