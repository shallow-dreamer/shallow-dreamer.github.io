---
layout:     post
title:      "el-drawer封装hooks测试"
subtitle:   ""
date:       2025-02-09
author:     " Shallow Dreamer"
header-img: "img/post-bg-js-version.jpg"
tags:
    - vue3
---

```
好的，用户需要将el-drawer组件二次封装成一个通用的hook，让它在任何组件中都能使用，可以自由控制弹出方向，并且允许添加自定义组件和事件。首先，我得回忆一下Element UI的el-drawer组件的用法和属性。el-drawer有direction属性控制方向，比如rtl、ltr、ttb、btt。然后，用户希望这个hook能返回一些方法，比如打开、关闭，以及可能的状态，比如是否显示。

接下来，我需要考虑如何用Vue 3的Composition API来封装。通常，hook会导出一些响应式变量和方法。所以，可能需要一个useDrawer的函数，返回open、close、toggle方法，以及方向direction、是否显示visible等状态。另外，用户需要在drawer内部添加自定义的内容和事件，所以应该使用插槽机制。但是hook本身不能直接处理模板，所以可能需要结合provide/inject或者render函数来处理。

不过，用户可能希望在使用这个hook的时候，能够在组件中直接写入自定义内容，而不是通过配置。这时候，可能需要将hook和组件一起封装。也就是说，创建一个自定义组件，比如DrawerComponent，然后在hook中控制这个组件的状态。这样，用户可以在自己的组件中使用hook提供的方法，并在模板里放置DrawerComponent，里面可以插入自定义内容。

那具体怎么设计呢？首先，hook负责管理状态，比如visible、direction、标题等。然后，DrawerComponent接收这些状态作为props，并渲染el-drawer。用户在使用时，先调用hook得到这些状态和方法，然后在模板里使用DrawerComponent，并通过插槽添加内容。这样，用户就可以在任意组件中引入DrawerComponent，并通过hook控制它的显示和方向。

接下来需要考虑如何让用户自定义事件。比如，用户可能在drawer内部有按钮，点击后触发事件。这时候，用户应该可以在插槽内直接绑定事件，就像普通组件一样。所以，DrawerComponent需要透传插槽，这样用户的内容和事件就能正常使用。

另外，可能还需要处理一些props，比如标题、宽度、是否显示关闭按钮等。这些可以通过hook的参数传入配置，或者在调用hook时动态设置。比如，hook可以接受一个配置对象，包含direction、title等默认值，同时允许在打开drawer时覆盖这些配置。

那具体的代码结构大概是怎样的呢？首先，定义useDrawer函数，里面用ref来管理visible、direction等状态。然后提供open、close、toggle方法。当调用open时，可以传入新的配置，比如改变方向或者标题。然后，DrawerComponent组件接收这些状态作为props，并绑定到el-drawer上。用户在使用时，引入DrawerComponent，并在模板中使用，同时通过插槽添加内容。

这样，用户的使用方式可能是这样的：

在组件中：

import { useDrawer } from './useDrawer';

const { Drawer, openDrawer } = useDrawer();

然后在模板里：

<Drawer>

<div @click="自定义事件">内容</div>

</Drawer>

不过，可能需要调整hook的结构，让Drawer成为一个组件，而hook返回这个组件和控制方法。或者，hook返回的是状态和方法，而用户需要自己引入Drawer组件，并将状态绑定上去。

或者，在hook内部，使用动态组件，并通过provide/inject来传递插槽内容。但这样可能比较复杂。

另一种方式是，将Drawer作为hook的一部分返回，用户需要在模板中显式渲染它，并通过作用域插槽或普通插槽插入内容。例如：

const { drawerProps, open, close } = useDrawer();

然后在模板中使用：

<DrawerComponent v-bind="drawerProps" @close="close">

<template #default>

用户自定义内容

</template>

</DrawerComponent>

这样，DrawerComponent是一个封装的el-drawer，接收相应的props和事件，并暴露插槽。这样用户就可以在插槽中添加任何内容和事件。

可能还需要处理默认插槽和作用域插槽的问题，确保用户可以灵活地添加内容。此外，还需要考虑如何在打开时动态修改配置，比如方向、标题等。比如，在调用open方法时，可以传入一个配置对象，更新这些属性。

综上，具体的步骤可能是：

1. 创建useDrawer hook，内部管理visible、direction、title等状态。

2. 提供open、close、toggle方法，允许在调用时传递配置参数来更新状态。

3. 创建一个DrawerComponent组件，接收这些状态作为props，并渲染el-drawer，同时暴露插槽。

4. 用户在使用时，调用useDrawer获取Drawer组件和控制方法，然后在模板中放置Drawer组件，并通过插槽添加内容。

现在，编写代码的大纲：

首先，useDrawer.ts：

import { ref } from 'vue';

export function useDrawer(defaultConfig = {}) {

const visible = ref(false);

const config = ref({

direction: 'rtl',

title: '默认标题',

size: '30%',

...defaultConfig,

});

const open = (overrideConfig = {}) => {

config.value = { ...config.value, ...overrideConfig };

visible.value = true;

};

const close = () => {

visible.value = false;

};

const toggle = () => {

visible.value = !visible.value;

};

return {

visible,

config,

open,

close,

toggle,

};

}

然后，DrawerComponent.vue：

<template>

<el-drawer

v-model="visible"

:direction="config.direction"

:title="config.title"

:size="config.size"

@close="close"

>

<slot></slot>

</el-drawer>

</template>

<script setup>

defineProps({

visible: Boolean,

config: Object,

close: Function,

});

</script>

不过，这样可能还不够灵活，因为用户需要将DrawerComponent引入到自己的组件中，并且绑定visible和config。这时候，或许更好的方式是将DrawerComponent和hook结合起来，让hook返回一个组件，或者通过provide/inject来传递状态。

或者，让useDrawer返回一个渲染函数，但这可能不太符合用户的使用习惯。

另一种方式是将DrawerComponent作为hook的一部分，用户在使用时，在setup中调用hook，然后在模板中使用返回的组件。例如：

在useDrawer中：

const drawerProps = computed(() => ({ visible: visible.value, ...config.value }));

return { drawerProps, open, close, toggle };

然后用户需要在模板中：

<DrawerComponent :visible="drawerProps.visible" :direction="drawerProps.direction" @close="close">

<!-- 插槽内容 -->

</DrawerComponent>

不过，这样用户需要自己处理props的绑定，可能不够方便。

或许更好的方式是，将DrawerComponent封装到hook内部，并返回一个组件，同时允许插槽内容。但如何在hook中处理插槽呢？这可能需要使用render函数或者jsx。

不过，在Vue 3中，可以使用defineComponent和setup函数来创建一个组件，然后通过作用域插槽传递数据。但这样可能会增加复杂度。

或者，采用另一种思路，将自定义内容通过函数式的方式传入。例如，在调用open方法时，传入一个渲染函数。但这样可能不够灵活，用户可能更习惯使用模板和插槽。

这时候，可能需要重新考虑设计。或许用户希望的是在任意组件中调用hook，然后在当前组件的模板中使用某个组件，该组件由hook提供，并且可以插入内容。例如，hook返回一个Drawer组件，用户可以在自己的模板中使用这个组件，并添加插槽内容。

不过，这在Vue中可能比较困难，因为组件的模板必须在template中编写，无法动态生成。所以，可能的解决方案是：用户需要显式地在模板中放置一个Drawer组件，并通过hook来控制它的状态。

例如，用户的使用方式应该是：

在script中：

const { drawerProps, open } = useDrawer();

在template中：

<MyDrawer v-bind="drawerProps">

<button @click="handleClick">自定义按钮</button>

</MyDrawer>

而MyDrawer组件内部封装了el-drawer，并接收drawerProps，同时暴露插槽。

这样，用户可以在自己的组件中引入MyDrawer，并通过hook控制它的显示和配置，同时在插槽中添加任何内容。

因此，在封装时，需要创建这个MyDrawer组件，并且在hook中管理状态。这样，用户调用hook得到状态和方法，同时在模板中使用MyDrawer组件，并绑定状态，插槽内容由用户提供。

所以，完整的代码结构可能如下：

useDrawer.js:

import { ref } from 'vue';

export function useDrawer(defaultConfig = {}) {

const visible = ref(false);

const config = ref({

direction: 'rtl',

title: '默认标题',

size: '30%',

...defaultConfig,

});

const open = (overrideConfig = {}) => {

config.value = { ...config.value, ...overrideConfig };

visible.value = true;

};

const close = () => {

visible.value = false;

};

const toggle = () => {

visible.value = !visible.value;

};

return {

Drawer: {

visible,

config,

open,

close,

toggle,

},

// 或者返回各个属性和方法

drawerProps: {

visible,

...config.value,

},

open,

close,

toggle,

};

}

然后，DrawerComponent.vue:

<script setup>

import { ElDrawer } from 'element-plus';

const props = defineProps({

visible: Boolean,

direction: {

type: String,

default: 'rtl',

},

title: String,

size: String,

});

const emit = defineEmits(['update:visible', 'close']);

const drawerVisible = computed({

get: () => props.visible,

set: (val) => emit('update:visible', val),

});

const handleClose = () => {

emit('close');

};

</script>

<template>

<ElDrawer

v-model="drawerVisible"

:direction="direction"

:title="title"

:size="size"

@close="handleClose"

>

<slot></slot>

</ElDrawer>

</template>

但这样的话，用户需要在他们的组件中引入DrawerComponent，并通过hook传递状态。例如：

用户组件：

<template>

<button @click="open({ direction: 'btt' })">打开抽屉</button>

<DrawerComponent

:visible="visible"

:direction="config.direction"

:title="config.title"

:size="config.size"

@close="close"

>

<!-- 自定义内容 -->

<button @click="submit">提交</button>

</DrawerComponent>

</template>

<script setup>

import { useDrawer } from './useDrawer';

import DrawerComponent from './DrawerComponent.vue';

const { visible, config, open, close } = useDrawer();

</script>

不过，这样用户需要手动绑定很多props，可能比较繁琐。所以，是否可以将DrawerComponent与hook更紧密地结合，让hook返回一个已经绑定好props的组件？

或者，在useDrawer中返回一个组件，用户可以直接在模板中使用，而不需要手动绑定props。例如：

const { Drawer, open, close } = useDrawer();

然后在模板中：

<Drawer>

内容

</Drawer>

但这样需要动态创建组件，可能比较复杂。在Vue中，可以使用createVNode和render函数，但可能会影响响应式。

另一种思路是，hook返回一个对象，其中包含需要绑定的props和事件，用户使用v-bind将其绑定到组件上。例如：

const { drawerProps, open, close } = useDrawer();

然后在模板中：

<DrawerComponent v-bind="drawerProps" @close="close">

...

</DrawerComponent>

这样，drawerProps是一个包含visible、direction等属性的响应式对象，用户只需要用v-bind绑定即可，而不需要逐个绑定。

这样，useDrawer可以返回drawerProps，其中包含visible和配置项，以及open、close等方法。DrawerComponent组件接收这些props，并绑定到el-drawer上。

这样，用户在组件中使用时，只需：

import { useDrawer } from './useDrawer';

import DrawerComponent from './DrawerComponent.vue';

const { drawerProps, open, close } = useDrawer();

然后在模板：

<DrawerComponent v-bind="drawerProps" @close="close">

<!-- 插槽内容 -->

</DrawerComponent>

这样，用户可以在任何组件中引入DrawerComponent，并通过hook控制它。同时，可以自定义插槽内容，添加事件等。

现在，调整useDrawer的实现：

在useDrawer函数中，管理visible和配置状态，并提供方法：

import { reactive, ref } from 'vue';

export function useDrawer(defaultConfig = {}) {

const visible = ref(false);

const config = reactive({

direction: 'rtl',

title: '默认标题',

size: '30%',

...defaultConfig,

});

const open = (overrideConfig = {}) => {

Object.assign(config, overrideConfig);

visible.value = true;

};

const close = () => {

visible.value = false;

};

const toggle = () => {

visible.value = !visible.value;

};

return {

drawerProps: {

visible: visible.value,

...config,

},

open,

close,

toggle,

};

}

不过，这里drawerProps.visible是visible.value的当前值，但visible是ref，所以应该用computed来保持响应式。否则，当visible变化时，drawerProps不会自动更新。

所以，更好的方式是将drawerProps包装成一个计算属性：

import { computed, reactive, ref } from 'vue';

export function useDrawer(defaultConfig = {}) {

const visible = ref(false);

const config = reactive({

direction: 'rtl',

title: '默认标题',

size: '30%',

...defaultConfig,

});

const drawerProps = computed(() => ({

visible: visible.value,

direction: config.direction,

title: config.title,

size: config.size,

}));

const open = (overrideConfig = {}) => {

Object.assign(config, overrideConfig);

visible.value = true;

};

// ...其他方法

return {

drawerProps,

open,

close,

toggle,

};

}

这样，drawerProps就是一个计算属性，当visible或config变化时，会自动更新。

然后，在DrawerComponent中，接收这些props：

<script setup>

defineProps({

visible: Boolean,

direction: String,

title: String,

size: String,

});

const emit = defineEmits(['update:visible', 'close']);

// 处理visible的双向绑定

const drawerVisible = computed({

get() {

return this.visible;

},

set(val) {

emit('update:visible', val);

},

});

const handleClose = () => {

emit('close');

};

</script>

<template>

<el-drawer

v-model="drawerVisible"

:direction="direction"

:title="title"

:size="size"

@close="handleClose"

>

<slot></slot>

</el-drawer>

</template>

用户在使用时，组件中的模板部分：

<template>

<button @click="open({ direction: 'ttb', title: '自定义标题' })">打开</button>

<DrawerComponent

v-bind="drawerProps"

@update:visible="(val) => (drawerProps.visible = val)"

@close="close"

>

<!-- 自定义内容 -->

<button @click="submit">提交</button>

</DrawerComponent>

</template>

不过，这里可能需要处理visible的双向绑定。因为drawerProps.visible是来自hook中的visible ref，所以当DrawerComponent内部修改visible时，应该更新hook中的visible值。可以通过在DrawerComponent中emit 'update:visible'，然后在父组件中监听这个事件，并更新visible.value。

但这样用户需要在模板中手动绑定@update:visible，可能不够方便。或许更好的方式是在hook的drawerProps中包含一个onUpdateVisible方法，或者在DrawerComponent内部直接修改hook中的visible ref。但这样可能破坏封装性。

或者，在useDrawer中返回的drawerProps包含visible的ref，并且DrawerComponent使用v-model:visible绑定。例如：

用户组件中的使用：

<DrawerComponent v-model:visible="drawerProps.visible" :direction="drawerProps.direction" ... >

但这样用户需要手动绑定每个属性，而无法简单地使用v-bind="drawerProps"。因为v-model需要单独的绑定。

可能这里存在一个矛盾：使用v-bind="drawerProps"会将visible作为prop传递，但更新visible需要emit事件。所以，在DrawerComponent中，visible应该是一个prop，并且通过v-model来同步。

但此时，drawerProps是一个计算属性，包含visible的值，而visible的更新需要在hook中处理。所以，当DrawerComponent内部emit update:visible事件时，用户需要监听这个事件并更新hook中的visible ref。

例如，在用户组件的模板中：

<DrawerComponent

v-bind="drawerProps"

@update:visible="val => visible.value = val"

@close="close"

>

但这样用户必须显式地处理这个事件，可能不够方便。或许可以在hook中提供一个方法，自动处理这个事件。例如，在hook返回的drawerProps中也包括事件处理器：

但这样可能比较复杂。或者，考虑在DrawerComponent内部直接修改visible的值，但这样会破坏单向数据流的原则，除非使用v-model。

可能更简单的方式是让用户手动处理visible的双向绑定。例如：

用户组件的模板：

<DrawerComponent

:visible="drawerProps.visible"

:direction="drawerProps.direction"

:title="drawerProps.title"

:size="drawerProps.size"

@update:visible="val => visible.value = val"

@close="close"

>

...

</DrawerComponent>

这样，当DrawerComponent内部的el-drawer的visible变化时，会触发update:visible事件，用户监听到后更新hook中的visible ref，从而触发重新渲染。

虽然这样需要用户手动绑定事件，但可能更符合Vue的数据流模式。或者，可以提供一个更简化的方式。

另一种思路是，在useDrawer中，返回的drawerProps包含visible的ref，而DrawerComponent内部使用v-model绑定visible。例如：

DrawerComponent的代码：

<el-drawer v-model="visible" ... >

但这样需要visible是一个ref，而用户传递的visible是来自hook的ref。这样，在DrawerComponent中，直接使用v-model绑定到props.visible可能可行，但需要确保props.visible是响应式的。

例如，用户在使用时，将visible作为ref传递给DrawerComponent的v-model：

<DrawerComponent v-model="visible" ... >

但这样用户需要从hook中获取visible的ref，而不是通过drawerProps。这可能与之前的封装方式不同。

综上，可能最直接的方式是让用户在模板中显式绑定visible，并使用v-model：

用户组件：

const { visible, direction, title, size, open, close } = useDrawer();

然后在模板中：

<DrawerComponent

v-model:visible="visible"

:direction="direction"

:title="title"

:size="size"

@close="close"

>

...

</DrawerComponent>

这样，用户需要从hook中获取各个属性，而不是通过drawerProps对象。这可能更灵活，但需要用户写更多的绑定代码。

或者，在hook中返回一个对象包含所有状态，用户可以使用展开运算符：

const drawer = useDrawer();

然后在模板中：

<DrawerComponent v-model:visible="drawer.visible" :direction="drawer.direction" ... >

这样，用户可以根据需要选择绑定的属性。

不过，回到用户的需求，他们希望可以方便地在任意组件中使用，并且可以自由控制方向，添加自定义组件和事件。所以，可能最佳的封装方式是：hook返回状态和方法，而用户负责将这些状态绑定到DrawerComponent组件，该组件封装了el-drawer，并暴露插槽。

这样，用户的使用步骤是：

1. 引入useDrawer和DrawerComponent。

2. 调用useDrawer获取状态和方法。

3. 在模板中使用DrawerComponent，绑定状态，并在插槽中添加内容和事件。

这样，用户可以在插槽中自由添加任何内容，包括带有自定义事件的组件。

最终，代码可能如下：

useDrawer.ts:

import { ref, reactive } from 'vue';

export function useDrawer(initialConfig = {}) {

const visible = ref(false);

const config = reactive({

direction: 'rtl',

title: '默认标题',

size: '30%',

...initialConfig,

});

const open = (overrideConfig = {}) => {

Object.assign(config, overrideConfig);

visible.value = true;

};

const close = () => {

visible.value = false;

};

const toggle = () => {

visible.value = !visible.value;

};

return {

visible,

config,

open,

close,

toggle,

};

}

DrawerComponent.vue:

<script setup>

import { ElDrawer } from 'element-plus';

const props = defineProps({

visible: {

type: Boolean,

required: true,

},

direction: {

type: String,

default: 'rtl',

},

title: String,

size: String,

});

const emit = defineEmits(['update:visible', 'close']);

const drawerVisible = computed({

get: () => props.visible,

set: (val) => emit('update:visible', val),

});

const handleClose = () => {

emit('close');

};

</script>

<template>

<ElDrawer

v-model="drawerVisible"

:direction="direction"

:title="title"

:size="size"

@close="handleClose"

>

<slot />

</ElDrawer>

</template>

用户的使用示例：

在某个组件中：

<template>

<button @click="open({ direction: 'btt' })">打开抽屉</button>

<DrawerComponent

v-model:visible="visible"

:direction="config.direction"

:title="config.title"

:size="config.size"

@close="close"

>

<div>

<h3>自定义内容</h3>

<button @click="handleSubmit">提交</button>

</div>

</DrawerComponent>

</template>

<script setup>

import { useDrawer } from './useDrawer';

import DrawerComponent from './DrawerComponent.vue';

const { visible, config, open, close } = useDrawer();

const handleSubmit = () => {

// 处理提交逻辑

close();

};

</script>

这样，用户可以通过调用open方法并传递配置来改变弹出方向和其他属性，同时在插槽中添加任意内容和事件。

总结，这种封装方式将状态管理与组件分离，用户通过hook获取状态和控制方法，并通过组件绑定状态，利用插槽添加内容，满足了灵活性和可重用性的需求。
```

下面是一个将 Element UI 的 el-drawer 封装为通用 Hook 的完整解决方案，包含可直接复用的代码：

```vue
<!-- DrawerComponent.vue -->
<template>
  <el-drawer
    v-model="drawerVisible"
    v-bind="$attrs"
    :direction="direction"
    @close="handleClose"
  >
    <slot />
  </el-drawer>
</template>

<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps({
  visible: {
    type: Boolean,
    required: true
  },
  direction: {
    type: String,
    default: 'rtl',
    validator: (val: string) => ['rtl', 'ltr', 'ttb', 'btt'].includes(val)
  }
})

const emit = defineEmits(['update:visible', 'close'])

const drawerVisible = computed({
  get: () => props.visible,
  set: (val) => emit('update:visible', val)
})

const handleClose = () => {
  emit('close')
}
</script>
```

```typescript
// useDrawer.ts
import { ref, reactive } from 'vue'

type DrawerDirection = 'rtl' | 'ltr' | 'ttb' | 'btt'

interface DrawerConfig {
  direction?: DrawerDirection
  title?: string
  size?: string | number
  withHeader?: boolean
  // 其他 el-drawer 支持的属性
}

export function useDrawer(initialConfig: DrawerConfig = {}) {
  const visible = ref(false)
  const config = reactive<DrawerConfig>({
    direction: 'rtl',
    title: '默认标题',
    size: '30%',
    withHeader: true,
    ...initialConfig
  })

  const open = (overrideConfig: DrawerConfig = {}) => {
    Object.assign(config, overrideConfig)
    visible.value = true
  }

  const close = () => {
    visible.value = false
  }

  const toggle = () => {
    visible.value = !visible.value
  }

  return {
    visible,
    config,
    open,
    close,
    toggle
  }
}
```

### 使用示例

```vue
<script setup lang="ts">
import { useDrawer } from './useDrawer'
import DrawerComponent from './DrawerComponent.vue'

const { visible, config, open, close } = useDrawer({
  direction: 'btt',
  title: '设置面板'
})

const handleSave = () => {
  console.log('保存操作')
  close()
}
</script>

<template>
  <el-button @click="open({ title: '编辑配置' })">打开抽屉</el-button>

  <DrawerComponent
    v-model:visible="visible"
    :direction="config.direction"
    :title="config.title"
    :size="config.size"
    @close="close"
  >
    <!-- 自定义内容 -->
    <div class="custom-content">
      <h3>个性化设置</h3>
      <el-form>
        <el-form-item label="主题颜色">
          <el-color-picker />
        </el-form-item>
        <el-form-item label="字体大小">
          <el-input-number />
        </el-form-item>
      </el-form>
      <el-button type="primary" @click="handleSave">保存设置</el-button>
    </div>
  </DrawerComponent>
</template>

<style scoped>
.custom-content {
  padding: 20px;
}
</style>
```

### 功能特性

1. **灵活控制**：通过 Hook 返回的 open/close/toggle 方法控制抽屉状态
2. **方向控制**：支持 4 种弹出方向（rtl/ltr/ttb/btt）
3. **动态配置**：open 方法支持传入临时配置覆盖默认设置
4. **完全可定制**：
   - 通过插槽可添加任意内容
   - 支持所有 el-drawer 原生属性和事件
   - 可通过 config 配置标题、尺寸等参数
5. **类型安全**：使用 TypeScript 编写，提供完善的类型提示

### 高级用法

```typescript
// 动态修改配置示例
const { open } = useDrawer()

// 打开时指定不同方向
const openFromTop = () => {
  open({
    direction: 'ttb',
    size: '50%',
    title: '从顶部弹出'
  })
}

// 自定义无标题抽屉
const openNoHeader = () => {
  open({
    withHeader: false,
    direction: 'btt',
    size: 200
  })
}
```

这个方案通过组合式 API 实现了抽屉组件的逻辑复用，同时保持视图层的灵活性。使用时可以自由选择：直接使用默认配置、通过初始化参数配置、或在调用 open 方法时动态覆盖配置。
