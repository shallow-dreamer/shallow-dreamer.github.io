---
layout:     post
title:      "利用button和input实现文件上传"
subtitle:   "使用js"
date:       2023-04-20
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - js
    - vue
---

代码：

```vue
<template>
	<button>
        导入文件
        <input type="file" @change="fileChange" accept=".*" :disable="disable"/>
    </button>
</template>
<script>
    data(){
        return{
            disable: false
        }
    }
    methods: {
        fileChange(event){
            // 文件处理函数
            let files = event.target.files
            let tempData = new FormData()
            for(let i = 0; i < files.length; i++){
                let file = files[i]
                tempData.append(file.name, file, file.name)
            }
            // 上传请求函数
            ...
        }
    }
</script>
<style>
    button {
        podition: relative;
        overflow: hidden;
        
        input {
            position: absolute;
            top: 0;
            left: 0;
            opacity: 0;
            width: 100%;
            height: 100%;
            font-size: 0;
            cursor: pointer;
        }
        button:disable {
            color: #ccc;
            background-color: #ccc;
            border-color: #aaa;
            cursor: not-allowed;
            
            input {
                cursor: not-allowed;
            }
        }
    }
</style>
```
