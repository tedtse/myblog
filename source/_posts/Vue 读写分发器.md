---
title: Vue 读写分发器 —— 编辑/详情 的合并解决方案
date: 2019-10-03 11:27:05
tags:
---

我们公司做的是 `to B` 的项目，框架选定的 `Vue` 。与互联网 `to C` 项目比，`to B` 以管理后台为主，没有太多复杂的交互，基本都是页面的增删改查，唯有一点就是量大！量大！量大！例如我们有一种场景需要实现表单的编辑功能。[场景地址](http://xiepeng.cc/rw-dispatcher/#/scene/edit)

### 场景

![编辑页](/assets/images/rw-dispatcher-edit.png)

很简单，这个页面用 `element-ui` 的表单组件就能完成。然后又需要一个表单的详情功能。

![详情页](/assets/images/rw-dispatcher-detail.png)

上面的编辑页和详情页，除了一个用来编辑，一个用来展示，其他都是一样的。编辑页用 `element-ui` 开发，详情页我们还需再开发一套，按照这种开发模式，就会有下面的问题：

### 问题

- 编码问题

编辑页需要写一遍，详情页也需要写一遍，即两遍。

- 样式问题

编辑页是现成的表单组件，不需要写样式，详情页需要重新写一遍，而且风格要和表单组件保持一致。

- 数量问题

一个前端项目大概有 **20 到 40** 组这样的页面，而一个管理系统由 **10** 个前端项目，10 个以上的后端项目组成。所以一个完整的管理系统会有 **200** 组以上编辑详情页面。

- 维护问题

呵呵，大家都懂的，需求会改动，设计也会改动。当表单项增一项或减一项，编辑页要改一遍，详情页也要改一遍；当表单布局从一行两列改为一行三列，编辑页要改一遍，详情页也要改一遍。

一两个页面还好，但是 `to B` 项目量大！量大！量大！维护起来就非常痛苦了。

### 解决方案

如果把它们写成一个文件会不会更好呢，有同事开始动手，将编辑页和详情页合并成一个文件，通过 `v-if` 来实现编辑和详情的判断，如下：

```
  <el-form ref="form" :model="form">
    <el-form-item label="活动名称">
      <template v-if="state='edit'">
        <el-input v-model="form.name" />
      </template>
      <template v-else>
        <div class="form-item--detail">
          <label>活动名称</label>
          <div class="form-item__content">{{ form.name }}</div>
        </div>
      </template>
    </el-form-item>
    ...
  </el-form>
```

结果这个文件的代码量剧增，可读性和维护性大大降低。

于是我针对表单组件写了一个扩展，暂时命名为 [组件分发器(传送门)](http://xiepeng.cc/rw-dispatcher/#/)，直接上代码。

以 `element-ui` 为例，表单（编辑页）我们是这样写的：
```
<template>
  <el-form ref="form" :model="form">
    <el-form-item label="活动名称">
      <el-input v-model="form.name" />
    </el-form-item>
    ...
  </el-form>
</template>
```

换成分发器可以这样写：
```
<template>
  <el-form ref="form" :model="form">
    <el-form-item label="活动名称">
      <el-input-dispatcher v-model="form.name" />
    </el-form-item>
    ...
  </el-form>
</template>

<script>
export default {
  provide () {
    return {
      rwDispatcherProvider: this
    }
  },
  data () {
    return {
      rwDispatcherState: 'write',
      ...
    }
  }
}
</script>
```

使用分发器比较使用表单只多了三步：

  1. 添加 provide 属性，其中 rwDispatcherProvider 的值指向自身

  2. data属性中添加 rwDispatcherState 做状态管理（read or write）

  3. 原来表单元素的标签加一个 -dispatcher 后缀，其它配置与表单组件保持不变

---

这样就完成了表单到分发器的代码迁移，我们这样理解它：

编辑页用表单组件实现，是`写`状态；

详情页不能编辑，所以是`读`状态。

那么，将表单组件替换成分发器，用状态参数管理读和写。如果是写状态，分发器将组件渲染成表单组件，如果读状态，分发器将组件渲染成详情组件。这样就实现了一套代码维护编辑和详情两个页面。

王婆卖瓜，自卖自夸。为什么要推荐分发器，主要它有以下几个特点：

- 支持多个组件库

`element-ui`、`iview` 都有对应的分发器。

- 使用简单

只多了三步。

- 完全定制化

提供多种配置，根据项目的实际需要自由定制渲染函数。

### 实现原理

分发器的实现原理是 `vue` 的 [provide/inject](https://cn.vuejs.org/v2/api/#provide-inject) 和 [渲染函数](https://cn.vuejs.org/v2/guide/render-function.html)。

各分发器根据 `provide` 的状态参数，进入对应的渲染函数，实现对应的渲染逻辑。有兴趣的同学可以去 `github` 上看看[源码](https://github.com/tedtse/rw-dispatcher-es)，如果觉得对自己有帮助的，欢迎 **start**（\*＾-＾\*），谢谢！

以下是其他相关链接：

[文档](http://xiepeng.cc/rw-dispatcher/#/)

[element-ui 分发器示例](https://github.com/tedtse/element-ui-rw-dispatcher-example)

[iview 分发器示例](https://github.com/tedtse/iview-rw-dispatcher-example)

### 不足与思考

渲染函数有两个参数 (h, context)，以 el-input-dispatcher 为例，它要传给 el-input 的[数据对象](https://cn.vuejs.org/v2/guide/render-function.html#%E6%B7%B1%E5%85%A5%E6%95%B0%E6%8D%AE%E5%AF%B9%E8%B1%A1) (class、style、attrs、props、on、directives、slot、ref 等等)，实现代码非常简单：
```
  ...
  render (h, context) {
    const { data, children } = context
    return h('el-input', data, children)
  },
  ...
```
很顺滑，所以我选择了渲染函数，但随着项目的深入，发现渲染函数有些问题，最大的问题是它没有生命周期，渲染的时候没有办法和其他组件通信，比如表单元素的 `size` 属性，可以设置在表单组件本身，如果本身没有设置会向上找 el-form-item 上的 size 属性，el-form-item 也没有会继续向上找 el-form。这个在 render 函数里是没有办法通过 context.parent 实现的。

或许[实例属性](https://cn.vuejs.org/v2/api/#%E5%AE%9E%E4%BE%8B%E5%B1%9E%E6%80%A7)可以尝试一下吧，vue 3.0 马上就要来了，到时再写一版吧，哈哈。