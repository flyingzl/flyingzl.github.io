---
title: Webpack中的require.context妙用
categories:
- 前端技术
date: 2018-07-08
tags: [Vue,Webpack]
---

## require.context简介

`require.context`是Webpack中用来管理依赖的一个函数，它的参数如下：
```javascript
require.context(directory, useSubdirectories = false, regExp = /^\.\//)
```

使用方式如下所示：
```javascript
// 创建一个test文件夹（不包含子目录）的上下文，可以require其下的所有js文件
const context = require.context("./test", false, /\.js$/)
const importAll = context => {
    // context.keys() 返回找到的js文件路径
    context.keys().forEach(key => context(key)
}
importAll()
```


## 实践 ---- 自动注册Vue组件
利用Vue开发时，我们常会抽离出一些组件并将其放到`components`文件夹，然后在Vue中进行引用，例如：
```html
<template>
    <search-input placeholder="搜索"></search-input>
    <dynamic-table></dynamic-table>
</template>
<script>
import SearchInput from '../components/SearchInput'
import DynamicTable from '../components/DynamicTable'
export default {
    name: 'App',
    components: {
        SearchInput,
        DynamicTable
    }
}
</script>
```

这是一种很好的代码复用方式，但是如果要引入的组件比较多时，一个一个引用会比较头疼，有没有一劳永逸的方法呢？有的，`require.context`可以上场了。 为了方便使用，我们可以写一个Vue插件

```javascript
// plugins/component.js
export default {

    install(Vue){
        const context = require.context('../components', true, /\.vue/)
        context.keys().forEach(key => {
            const component = context(key).default
            Vue.component(component.name, component)
        })
    }
}

// app.js
import Vue from 'vue'
import globalComponents from './plugins/components'
Vue.use(globalComponents)

```

原理很简单，我们拿到组件后，通过`Vue.component`进行全局注册。这样注册后，我们可以在Vue中的template里面直接引入组件了。怎么样，是不是方便多啦？


## 实践 ---- 简化Vuex开发
首先上一张Vuex的架构图
![](https://raw.githubusercontent.com/vuejs/vuex/dev/docs/.vuepress/public/vuex.png)

从架构图上可以看到，我们开发时，会有 `action`、`muations`以及`store`三个部分，开发时，我们会将这三个部分进行拆开，会有如下的文件夹结构：

```bash
.
├── actions
│   ├── index.js
│   ├── overview.js
│   └── settings.js
├── mutations
│   ├── index.js
│   ├── overview.js
│   └── settings.js
└── store
    └── index.js
```

`actions`文件下的`index.js`用来引用其他js文件，例如

```javascript
// actions/index.js

import overviewActions from './overview'
import settingsActions from './settings'
export default {
    ...overviewActions
    ...settingsActions
}

```

`muations`文件下的`index.js`用来引用其他js文件，例如:

```javascript
// muations/index.js

import overview from './overview'
import settings from './settings'
export default {
    overview,
    settings
}

```

`store`文件夹下的`index.js`用来将`action`和`mutation`进行载入：
```javascript
import Vue from "vue"
import Vuex from "vuex"
import actions from "../actions"
import mutations from "../mutations"

Vue.use(Vuex)

Vue.config.devtools = __DEV__

!__DEV__ && (Vue.config.errorHandler = function (err, vm) {
    vm.$store.dispatch("hidePageLoading")
    console.error(err.message)
})

export default new Vuex.Store({
    actions,
    modules: mutations,
    strict: false
})

```

这样代码结构很清晰，但是大家是否意识到，每次我们增加一个模块，都需修改`actions`以及`mutations`下的`index.js`, 很烦有没有？git提交时偶尔还发生冲突，很不爽哇！怎么解决？`require.context`来帮忙~~

我们现在删除`actions`以及`mutations`下的`index.js`文件，然后修改`store/index.js`：

```javascript
import Vue from "vue"
import Vuex from "vuex"

Vue.use(Vuex)

Vue.config.devtools = __DEV__

!__DEV__ && (Vue.config.errorHandler = function (err, vm) {
    vm.$store.dispatch("hidePageLoading")
    console.error(err.message)
});

// 自动引入actions下的js文件，将文件的各个对象合并为一个大对象
const actionContext = require.context('../actions', false, /.*\.js/)
const actions = actionContext
    .keys()
    .reduce((prev, cur) => ({ ...prev, ...actionContext(cur).default }), {})

// 自动引入mutation文件下的js文件，以文件名字作为对象的key
const mutationContext = require.context('../mutations', false, /.*\.js/)
const modules = mutationContext.keys().reduce((prev, cur) => {
    const key = cur.match(/(\w+)\.js/)[1]
    prev[key] = mutationContext(cur).default
    return prev
}, {})

export default new Vuex.Store({
    actions,
    modules,
    strict: false
})

```

这样处理后，我们以后增加一个模块，直接往`actions`和`muations`下直接加js文件就好了，大家互不干扰，快乐协作！


## 总结
以上是我开发中总结的一些小技巧，希望对大家有所帮助：-）
