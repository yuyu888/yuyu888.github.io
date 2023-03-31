---
layout: mypost
title: Vue自定义组件使用v-model
categories: [VUE]
---

## v-model基本概念

v-model 实际上就是 $emit('input') 以及 props:value 的组合语法糖，只要组件中满足这两个条件，就可以在组件中使用 v-model。 

按照Vue官网介绍，Vue的双向绑定是一种语法糖。本质上是
>“负责监听用户的输入事件以更新数据，并对一些极端场景进行一些特殊处理。”


````js
<input v-model="thisValue">
````

等价于：

````js
<input
    :value="thisValue"
    @input="thisValue = $event.target.value"
>
````

为什么说上述的原始写法只针对于input 的text呢？
vue官网解释如下

>v-model 在内部为不同的输入元素使用不同的属性并抛出不同的事件：
> + text 和 textarea 元素使用 value 属性和 input 事件；  
> + checkbox 和 radio 使用 checked 属性和 change 事件；  
> + select 字段将 value 作为 prop 并将 change 作为事件。  

无论是任何组件，都可以实现 v-model。

而实现 v-model 的要点，主要就是以下几点：  
+ **props:value** 用来控制 v-model 所绑定的值。  
+ **currentValue** 由于 单向数据流 的原因，需要使用 currentValue 避免子组件对于 props 的直接操作。  
+ **$emit('input')** 用来控制 v-model 值的修改操作,所有对于 props 值的修改，都要通知父组件。  
+ **watch监听** 当组件初始化时从 value 获取一次值，并且当父组件直接修改 v-model 绑定值的时候，对于 value 的及时监听。

有时候 不想用 props:value 以及 $emit('input') ，我想换一个名字，那么此时， model 可以帮你实现。

因为这两个名字在一些原生表单元素里，有其它用处。

````
export default {
  model: {
    prop: 'number',
    event: 'change'
  },
}
````

这种情况下，那就是使用 props:number 以及 $emit('change')。


## 实例说明一切

### 下拉菜单

**调用**
````js
        <fruit-selector v-model="fruit"></fruit-selector>

````

**组件**
````js
<template>
  <el-select
    v-model="selectedValue"
    filterable
    placeholder="请选择水果"
    clearable
    :disabled="disabled"
    @change="changeHandler">
    <el-option
      ref="fruitOptions"
      v-for="item in fruitOptions"
      :key="item.value"
      :label="item.label"
      :value="item.value">
    </el-option>
  </el-select>
</template>

<script>
  export default {
    name: 'fruit-selector',
    components: {},
    props: {
      disabled: false,
      value: ''
    },
    data () {
      return {
        fruitOptions: [
          {'value': '1', 'title': '苹果', 'label': '苹果'},
          {'value': '2', 'title': '草莓', 'label': '草莓'},
          {'value': '3', 'title': '香蕉', 'label': '香蕉'}
        ],
        selectedValue: ''
      }
    },
    watch: {
      value () {
        this.selectedValue = this.value
      }
    },
    mounted () {
      this.init()
    },
    created () {
      this.selectedValue = this.value
    },
    methods: {
      init () {
      },
      changeHandler () {
        this.$emit('input', this.selectedValue)
      }
    }
  }
</script>

````

