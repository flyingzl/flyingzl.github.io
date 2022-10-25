---
title: 记一次Vue中的v-for踩坑之旅
date: 2018-07-20 17:30:00
categories:
- 前端技术
tags: Vue
---

用过Vue的同学都知道，`v-for`指令常用于遍历数组或者对象，然后依次渲染出指定的内容。同时，我们也知道，官方文档也建议，在使用`v-for`指令时，记得要加上`key`属性，方便提升应用性能。例如一个简单的增删Todo应用如下所示：

```html
<template>
    <div class="todos">
        <input v-model.trim="task" @keypress.enter="onSaveTodo" placeholder="请输入待办任务" />
        <ul>
            <li v-for="(todo, index) in todos" :key="index">
                <span @click="onRemoveTodo(index)">{{ todo }} <i class="remove">&times;</i></span>
            </li>
        </ul>
    </div>
</template>

<script>
    export default {
        name: 'TodoApp',
        data() {
            return {
                todos: [],
                task: ''
            }
        },
        methods: {
            onSaveTodo() {
                this.todos.push(this.task)
                this.task = ''
            },
            onRemoveTodo(index) {
                this.todos.splice(index, 1)
            }
        }
    }
</script>

<style>
.todos .remove {
    color: #ff0000;
    cursor: pointer;
}
</style>
```


{% raw %}
<p data-height="400" data-theme-id="0" data-slug-hash="wjNxoP" data-default-tab="js" data-user="flyingzl" data-embed-version="2" data-pen-title="vue-v-for" data-preview="true" class="codepen">See the Pen <a href="https://codepen.io/flyingzl/pen/wjNxoP/">vue中v-for使用</a></p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>
{% endraw %}

代码很简单明了，也运行的很高效。我们用了`v-for`指令，也加了`key`, 一切都和完美，感叹Vue真好用，真是高效哇！


## 组件封装

在Vue中，官方建议我们多进行组件封装和抽象，这样方便后期维护。因为每一个Todo都有自己的状态，例如完成或者未完成， 我们需要将每一个Todo抽象为组件。所以我们要做一下简单的改进：新建一个`TodoItem.vue`，然后在主文件中导入使用

```html
<template>
    <div class="todo-item" :class="{checked: checked}">
        <input type="checkbox" v-model="checked" />
        <span class="label" @click="onRemoveItem">{{ todo }} <i class="remove">&times;</i></span>
    </div>
</template>

<script>
export default {
    name: 'TodoItem'
    props: {
        todo: String
    },
    data() {
        return {
            checked: false
        }
    },
    methods: {
        onRemoveItem() {
            this.$emit('remove')
        }
    }
}
</script>

<style>
    .todo-item.checked .label {
        text-decoration: line-through;
    }
</style>
```

代码非常简单，我们在`TodoItem.vue`中新增加了一个`checked`属性，当复选框勾选后，todo文字会显示删除效果。重新修改下主文件

```html
<template>
    <div class="todos">
        <input v-model.trim="task" @keypress.enter="onSaveTodo" placeholder="请输入待办任务" />
        <ul>
            <li v-for="(todo, index) in todos" :key="index">
                <todo-item :todo="todo" @remove="onRemoveTodo(index)"></todo-item>
            </li>
        </ul>
    </div>
</template>

<script>
    import TodoItem from './TodoItem'
    export default {
        name: 'TodoApp',
        components: {
            TodoItem
        },
        data() {
            return {
                todos: [],
                task: ''
            }
        },
        methods: {
            onSaveTodo() {
                this.todos.push(this.task)
                this.task = ''
            },

            onRemoveTodo(index) {
                this.todos.splice(index, 1)
            }
        }
    }
</script>

<style>
.todos .remove {
    color: #ff0000;
    cursor: pointer;
}
</style>
```
代码变化不大，只是在`v-for`循环时加入了`<todo-item></todo-item>`而已。看看运行效果

{% raw %}
<p data-height="500" data-theme-id="0" data-slug-hash="JBbaeR" data-default-tab="js,result" data-user="flyingzl" data-embed-version="2" data-pen-title="vue中v-for使用（组建嵌套）" data-preview="true" class="codepen">See the Pen <a href="https://codepen.io/flyingzl/pen/JBbaeR/">vue中v-for使用（组建嵌套）</a></p>
{% endraw %}

![](https://g.asyncoder.com/20180720-01.png)

好像没啥问题？ 但是如果我们增加两条数据，将第一条记录勾选后然后再删除，令人费解的事情发生了：

![](https://g.asyncoder.com/20180720-02.png)

可以清晰地看到，第二条记录之前是未勾选状态，但是删除第一条后，它变成了勾选状态？这是为什么呢？


## 问题分析

这个问题其实我现实项目中的一个抽象，当时我也遇到了类似的问题，想了好几个小时都没解决。我一行一行分析我的代码，是不是代码哪里写错了？最后一行一行分析，突然想到是不是`key`用的不对？于是我将`key`弄成一个唯一的id，然后奇迹发生了，页面都正常了。这是为什么呢？

在我们的例子，如果我们我们将`v-for`中的`key`改成如下所示（保证todo不重复）问题就解决了：

```html
<ul>
    <!-- 为了演示访问表，假设todo是永不相同的  -->
    <li v-for="(todo, index) in todos" :key="todo">
        <span @click="onRemoveTodo(index)">{{ todo }} <i class="remove">&times;</i></span>
    </li>
</ul>
```

虽然当时问题是解决了，但是这个`v-for`的问题一直在困扰我，到底是什么原因导致这种现象发生，为什么`key`弄成唯一的id就好使了呢？官方说在`v-for`时增加`key`可以提升应用性能，到底是怎么提升的？ 

刚好最近在看Vue Virtual DOM的diff算法，终于从中间找到了解决该问题的曙光


## Vue中的Virtual DOM

在Vue中，`template`中的内容最后都会被解析并渲染为[VNode](https://github.com/answershuto/learnVue/blob/master/docs/VNode%E8%8A%82%E7%82%B9.MarkDown), 这个就是所谓的Virtual DOM。当我们修改Vue中的数据后，Vue会对前后两次的VNode进行diff，找出最小的差异，然后再渲染DOM，这样可以提高应用的性能。

VNode其实是对真实DOM的Javascript抽象，例如一个简答的DOM树如下所示：

```html
<div class="test">
    <span class="demo">hello,VNode</span>
</div>
```

在VNode中，会这样进行展示：

```javascript
{
    tag: 'div'
    data: {
        class: 'test'
    },
    children: [
        {
            tag: 'span',
            data: {
                class: 'demo'
            }
            text: 'hello,VNode'
        }
    ]
}
```

也就是说，VNode可以真实描述并还原DOM。


## Virtual DOM diff算法

> Vue中的Virtual DOM diff算法比较复杂，一言两语无法描述清楚。由于网上已经有很多文章，我这里只针对`v-for`问题进行针对性解释。

我们先看看diff的核心函数（这些是源码的抽象，源码里面更为复杂）：

```javascript
function patch (oldVNode, vnode, parentElm) {
    if (!oldVnode) {
        addVnodes(parentElm, null, vnode, 0, vnode.length - 1);
    } else if (!vnode) {
        removeVnodes(parentElm, oldVnode, 0, oldVnode.length - 1);
    } else {
        if (sameVnode(oldVNode, vnode)) {
            patchVnode(oldVNode, vnode);
        } else {
            removeVnodes(parentElm, oldVnode, 0, oldVnode.length - 1);
            addVnodes(parentElm, null, vnode, 0, vnode.length - 1);
        }
    }
}
```

`patch`就是比较前后两个`VNode`，然后找出其最小差异并修改、创建或者删除DOM。在该函数中，`oldVNode`代表旧的数据，`vnode`代表最新的数据。比较时，会进行深度优先逐层进行比较。如下图所示：

![](https://g.asyncoder.com/20180720-04.png)
![](https://g.asyncoder.com/20180720-05.png)

也就是说，在上图中，只有相同颜色的VNode才进行比较，这样算法复杂度就比较低，整体下来只有O(n)，效率算法非常高了。

从`patch`函数可以看出，diff算法的的核心逻辑是这样的：

- 如果旧的VNode不存在，新的VNode存在，则创建新的DOM
- 如果旧的VNode存在，新的VNode不存在，则删除旧的DOM
- 如果新旧两个VNode都存在并相同，则找出最小差异然后更新DOM
- 如果新旧两个VNode都存但不相同，则将旧的DOM删除，然后创建新的DOM

这里的关键是：如何判断两个`VNode`相同呢？请看下面的代码：

```javascript
function sameInputType (a, b) {
    if (a.tag !== 'input') return true
    let i
    const typeA = (i = a.data) && (i = i.attrs) && i.type
    const typeB = (i = b.data) && (i = i.attrs) && i.type
    return typeA === typeB
}

function sameVnode () {
    return (
        a.key === b.key &&
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        !!a.data === !!b.data &&
        sameInputType(a, b)
    )
}
```

也就是说，只有当 key、 tag、 isComment（是否为注释节点）相同、 data同时定义（或不定义），同时满足当标签类型为input的时候type相同，那么它们就是相同的VNode。

注意这里的**key**相同，才代表VNode相同。对比我们之前出错的样例，因为我们的key是索引号，可知第一条记录的索引号为0。当第一条记录被删除后，第二条记录的key的索引号会从1变为0，这样导致了两者的key相同。因为key相同时，diff算法会认为它们是相同的VNode，那么旧的VNode（如果VNode是一个组件，它有一个componentInstance指向Vue实例）指向的Vue实例会被复用，导致显示出错。修改key为唯一id时，根据上文`patch`函数的逻辑，旧的VNode所对应的DOM会被干掉，然后得新的DOM会被创建。因为是新创建的DOM，那么对应的Vue也是新创建的，一切就会显示正常。

所以，保证`key`唯一，就可以解决组件出错的问题。

上文中，我们没有提到diff算法的核心，也就是说当两个VNode相同时，`patchVnode`是怎么实现的。建议大家阅读相关参考文章。


## 总结

`v-for`使用非常简单，但是要特别注意`key`的使用。官方之所以说加上`key`会提升应用性能是因为：`key`相同时，两个VNode会相同，可以避免不必要的DOM更新。而且在diff内部，也会根据`key`来跟踪VNode。但是，官方也说了，尽量保证`key`是唯一的id，这样可以避免一些匪夷所思的bug。


## 参考资料

- [VirtualDOM与diff(Vue实现).MarkDown](https://github.com/answershuto/learnVue/blob/master/docs/VirtualDOM%E4%B8%8Ediff(Vue%E5%AE%9E%E7%8E%B0).MarkDown)

- [数据状态更新时的差异 diff 及 patch 机制](https://github.com/answershuto/VueDemo/blob/master/%E3%80%8A%E6%95%B0%E6%8D%AE%E7%8A%B6%E6%80%81%E6%9B%B4%E6%96%B0%E6%97%B6%E7%9A%84%E5%B7%AE%E5%BC%82%20diff%20%E5%8F%8A%20patch%20%E6%9C%BA%E5%88%B6%E3%80%8B.js)

- [解析vue2.0的diff算法](https://segmentfault.com/a/1190000008782928)

- [详解vue的diff算法](https://www.cnblogs.com/wind-lanyan/p/9061684.html)

