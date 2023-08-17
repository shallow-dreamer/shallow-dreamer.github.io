---
layout:     post
title:      "webpack多项目代理及axios嵌套"
subtitle:   ""
date:       2023-08-17
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - vue
    - webpack
    - axios
---

在Vue 3 中实现 Axios 的封装嵌套，以及在主项目的 Webpack 中将不同的子项目代理（proxy）到不同的服务器，可以分为以下几个步骤：

## 封装 Axios

1. 首先，创建一个独立的文件，比如 `api.js`，用于封装 Axios：

```javascript
// api.js

import axios from 'axios';

const instance = axios.create({
  baseURL: process.env.VUE_APP_BASE_API, // Set your base API URL
  timeout: 10000, // Set the request timeout
});

const request = (config) => {
  return instance(config)
    .then((response) => {
      return response.data;
    })
    .catch((error) => {
      throw error;
    });
};

export default request;
```

2. 在需要使用 Axios 的地方引入并使用封装的函数：

```javascript
import request from './api';

// Example usage:
request({
  url: '/endpoint',
  method: 'get',
  // ...other config options
}).then((data) => {
  // Handle the response data
}).catch((error) => {
  // Handle the error
});
```

## 子项目代理设置

假设你的主项目中有多个子项目，每个子项目需要代理到不同的服务器。你可以通过 Vue CLI 的配置来实现这一点。

1. 在主项目的根目录下，创建一个 `vue.config.js` 文件（如果没有的话）：

```javascript
// vue.config.js

module.exports = {
  devServer: {
    proxy: {
      // Example proxy configuration for Subproject A
      '/subproject-a': {
        target: 'http://subproject-a-server-url',
        changeOrigin: true,
        // ...other proxy options
      },

      // Example proxy configuration for Subproject B
      '/subproject-b': {
        target: 'http://subproject-b-server-url',
        changeOrigin: true,
        // ...other proxy options
      },
      
      // Add more proxy configurations for other subprojects as needed
    }
  }
};
```

2. 在子项目中，你可以将 API 请求的 URL 改为相对路径，比如 `/subproject-a/endpoint` 或 `/subproject-b/endpoint`，然后 Vue CLI 会自动将这些请求代理到对应的服务器。

```javascript
// In a component within Subproject A
import request from './api';

request({
  url: '/subproject-a/endpoint',
  method: 'get',
  // ...other config options
}).then((data) => {
  // Handle the response data
}).catch((error) => {
  // Handle the error
});
```

注意：这里的示例假设你已经创建了各子项目的配置和代理，并且在不同的子项目中使用了不同的 Axios 实例（比如在 `api.js` 中设置了不同的 `baseURL`）。

这些步骤可以帮助你实现 Axios 的封装和子项目的代理设置，使你能够更好地管理多个子项目的 API 请求和代理需求。具体的配置和路径根据你的项目结构和需求进行调整。

在Vue 3中，你可以通过以下步骤来实现对Axios的子项目底层封装并嵌套在主项目的Axios封装中，以及如何在Webpack中将不同的子项目代理到不同的服务器。

**1. 子项目的底层封装**

首先，为每个子项目创建一个独立的Axios实例。可以在每个子项目中创建一个单独的文件来处理这些封装逻辑，例如`api.js`：

```javascript
// 子项目1的api.js
import axios from 'axios';

const instance = axios.create({
  baseURL: 'https://subproject1-api-url.com',
  // 其他配置项
});

export default instance;
```

你可以为其他子项目创建类似的`api.js`文件，然后在每个子项目中引入并使用这些独立的Axios实例。

**2. 主项目的Axios封装**

在主项目中，你可以创建一个统一的Axios封装，将子项目的Axios实例作为其中的一部分。可以在主项目中的`axios.js`文件中执行以下操作：

```javascript
import axios from 'axios';
import subProject1Api from './subprojects/subproject1/api'; // 导入子项目1的Axios实例
// 导入其他子项目的Axios实例

const instance = axios.create({
  baseURL: 'https://main-project-api-url.com',
  // 其他配置项
});

// 将子项目的Axios实例添加到主项目的Axios实例中
instance.subProject1 = subProject1Api;
// 将其他子项目的Axios实例也添加到主项目的Axios实例中

export default instance;
```

这样，你可以在主项目中使用`axios`来访问主项目的接口，同时也可以通过`axios.subProject1`来访问子项目1的接口，以此类推。

**3. Webpack中的代理配置**

假设你想要将不同的子项目代理到不同的服务器。你可以在主项目的`vue.config.js`文件中配置Webpack的代理选项来实现这一点：

```javascript
module.exports = {
  devServer: {
    proxy: {
      '/api-subproject1': {
        target: 'https://subproject1-api-url.com',
        // 其他代理配置项
      },
      // 添加其他子项目的代理配置
    }
  }
};
```

上述配置中，`/api-subproject1`是你在主项目中发起请求时的路径前缀，当请求路径以这个前缀开始时，Webpack会将请求代理到指定的子项目服务器。

注意，不同的子项目代理配置需要根据子项目的实际情况进行设置，包括`target`和其他代理配置项。

通过以上步骤，你可以在Vue 3项目中实现Axios的子项目底层封装并嵌套在主项目的Axios封装中，并且可以使用Webpack的代理配置来将不同的子项目代理到不同的服务器。

在 Vue 3 中，你可以通过以下步骤来实现 Axios 的子项目底层封装并嵌套在主项目的 Axios 封装中，以及在主项目的 Webpack 配置中将不同的子项目代理到不同的服务器。

**步骤一：子项目底层封装**

首先，你需要为子项目单独创建一个 Axios 实例，然后对其进行封装，以便在主项目中使用。假设你的子项目名为 SubProject。

1. 在子项目的代码中创建一个 `axiosInstance.js` 文件，用于创建子项目的 Axios 实例：

```javascript
// src/subproject/axiosInstance.js

import axios from 'axios';

const axiosInstance = axios.create({
  baseURL: '/subproject-api', // You can adjust this baseURL as needed
});

export default axiosInstance;
```

2. 在主项目的代码中创建一个 `subproject` 文件夹，将子项目的 `axiosInstance.js` 放入其中。

**步骤二：主项目中的 Axios 封装**

1. 在主项目中的 `src` 文件夹下创建一个 `axiosInstance.js` 文件，用于封装主项目的 Axios 实例，同时引入子项目的 Axios 实例：

```javascript
// src/axiosInstance.js

import axios from 'axios';
import subprojectAxios from './subproject/axiosInstance';

const axiosInstance = axios.create({
  baseURL: '/mainproject-api', // You can adjust this baseURL as needed
});

const mainAxios = {
  instance: axiosInstance,
  subproject: subprojectAxios,
};

export default mainAxios;
```

**步骤三：Webpack 配置代理**

假设你想将主项目的 API 请求代理到 `http://localhost:3000`，而将子项目的 API 请求代理到 `http://localhost:4000`。

1. 打开主项目的 `vue.config.js` 文件，添加以下代理配置：

```javascript
// vue.config.js

module.exports = {
  devServer: {
    proxy: {
      '/mainproject-api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
        pathRewrite: {
          '^/mainproject-api': '',
        },
      },
      '/subproject-api': {
        target: 'http://localhost:4000',
        changeOrigin: true,
        pathRewrite: {
          '^/subproject-api': '',
        },
      },
    },
  },
};
```

这样，当你在主项目中发起 `/mainproject-api` 的请求时，它会被代理到 `http://localhost:3000`，而发起 `/subproject-api` 的请求时，会被代理到 `http://localhost:4000`。

现在你已经完成了在 Vue 3 中实现 Axios 的子项目底层封装并嵌套在主项目的 Axios 封装中，以及使用 Webpack 配置将不同的子项目代理到不同的服务器的过程。
