---
layout:     post
title:      "在多个页面中使用多个类似的图表"
subtitle:   "项目问题记录"
date:       2023-04-19
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - 项目问题记录
    - echarts
    - vue
---

###### 在多个页面中使用多个类似的图表

遇到的问题：

1、图表在页面切换时请求结束需要操作dom，所以为组件添加保活机制

2、多个图表在一个页面中时，因为echarts是根据dom元素的id来初始化的，所以需要给组件赋予不同的id

3、在保活页面周期对图表进行绘制



demo代码

多个父组件：为了解决第二个问题，在父组件向子组件的传值中添加了一个mark标记，并由子组件使用mark标记合成id值

```vue
<!-- 父组件有多个子组件 -->
<div class="chart chartOne">
    <keep-alive>
    	<chart-one :data="chartDataOne" v-if="flag"></chart-one>
    </keep-alive>
</div>
<div class="chart chartOne">
    <keep-alive>
    	<chart-one :data="chartDataOne" v-if="flag"></chart-one>
    </keep-alive>
</div>
<div class="chart chartOne">
    <keep-alive>
    	<chart-one :data="chartDataOne" v-if="flag"></chart-one>
    </keep-alive>
</div>
import { reactive } from 'vue'
<script>
 export default{
     setup(){
         const data = reactive({
             //图表更新flag
             flag: true,
             loadingFlag: false,
             //图表数据
             chartDataOne: {
                 mark: 'one',
                 data: []
             },
             chartDataTwo: {
                 mark: 'two',
                 data: []
             },
             chartDataThree: {
                 mark: 'three',
                 data: []
             }
         }
	})
}
</script>
```

多个类似子组件

```vue
<template>
	<div :id="'data.mark' + 'one'" class="one"></div>
</template>
<script>
import * as echarts from 'echarts'
import { onMounted, reactive, onActivated, ref, markRaw } from 'vue'
export default {
    props: ['data'],
    setup(props) {
        let chartOne = null
        let option = reactive({
            //图表配置
            ...
        })
        onActivated(() => {
            if(chartOne){
                setTimeout(() => {
                    chartOne.resize()
                })
            }
            drawChart()
        })
        function drawChart(){
            let ChartDom = document.getElementById(`${props.data.mark}one`)
            chartOne = markRaw(echarts.init(chartDom))
            let data = props.data.data
            //图表宽度自适应
            window.addEventListener('resize', function(){
                chartOne.resize()
            })
            for(let i = 0; i < option.series.length; i++){
                option.series[i].data = data[i]
            }
            option && chartOne.setOption(option)
        }
    }
}
</script>
<style lang="less" scoped>
.one {
    width: 100%;
    height: 100%;
}
</style>
```

第三个问题：在子组件中调用onActivated钩子函数，在钩子函数中调用绘图函数

思路总结：在父组件中调用多个图表组件，并且为组件添加保活机制，是组件在切换页面时可以正常接受后端返回的数据，在向子组件传的值中添加一个标记，防止多个父组件图表id名重复，导致最后的图表绘制出现冲突，比如两个一样的图表数据不一样，但它都会绘制最后返回的数据，在子组件中使用接收到的数据中的标记合成每个图表独有的id，再获取这个id进行echarts图表初始化，最后编写图表绘制函数，并在onActivated钩子函数中使用
