---
title: vue3组件通信的方式
date: 2021-09-24 22:36:20
author: Jack-zhang
categories: vue
tags:
   - vue3
   - JS
summary: 关于vue3组件间通信的应用,兼容vue2
---
## 父传子的方式

### 使用props

* 关于props
  * Props是你可以在组件上注册一些自定义的attribute；
  * 父组件给这些attribute赋值，子组件通过attribute的名称获取到对应的值

* 语法
  * 字符串数组:数组中的字符串就是`attribute`的名称

  ```js
  //父组件传递数据
  <Demo :name="xxx"/>//可以动态的绑定
  
  //子组件接收数据
  props:["name"]
  ```

  * 对象类型，对象类型我们可以在指定attribute名称的同时，可以传入限制
    * 指定传入的<span style="color:red">attribute的类型(type)</span>
    * 指定传入的<span style="color:red">attribute是否是必传的(require)</span>
    * 指定没有传入时，<span style="color:red">attribute的默认值(default)</span>

  ```js
  //父组件传递数据
  <Demo id="123" class="aaa" tittle="hhh" content="ggg" :a="a">
  
  //子组件接收父组件数据
  const props= defineProps<{
    tittle: String,
    content: String,
    a: {
      type: String,
      require: true,
      default: "111",
    }>()
  ```

* type类型
  * String
  * Number
  * Boolean
  * Array
  * Object
  * Date
  * Function
  * Symbol

* 注意点:类型有关对象的,默认值要写成一个工厂函数(例如`Array`,`Object`,`Date`,`Function`)<span style="color:red">建议全写成函数</span>

>vue3的顶层setup中使用`defineprops`如果使用的是泛型,不能设置默认值,如果使用定义值的写法,有默认值,但比较繁琐

* 并且如果使用泛型,ts会对其进行检查,但是自定义值不会

```ts
//使用泛型的写法
interface test {
  test: number
  test2: Object
}
const prop = defineProps<{
  msg: { type: string }
  obj: test
}>()
//使用自定义值
const prop = defineProps({
  msg: String,
  obj: {
    type: Object,
    default: () => ({})
  }
})
```

### 非Prop的Attribute

* 定义
  * 当我们传递给一个组件某个属性，但是该属性并没有定义对应的props或者emits时，就称之为 非Prop的Attribute
  * 常见的包括`class`,`style`,`id属性`等

* **Attribute继承**
  * 当组件有单个根节点时，非Prop的Attribute将自动添加到根节点的Attribute中
* **注意**:
  1. 如果我们**不希望组件的根元素继承attribute**，可以在组件中设置 **inheritAttrs: false**
     * <span style="color:red">但是`class` , `style`</span>，不是 的一部分`$attrs`，仍将应用于组件的根元素
  2. 通过 `$attrs`来访问所有的 非props的attribute
  3. 使用`v-bind`可以直接解构绑定的对象,而不需要一个个单独传递
  
     ```html
     //父组件传值
     <my-component id="my-id" class="my-class"></my-component>
   
     //子组件取值
     <template>
       <label>
        <input type="text" v-bind="$attrs" />
        <!-- 访问所有的属性,且class属性绑定到根节点label上 -->
        <input type="text" :id="$attrs.id" />
        <!-- 拿到id属性 -->
       </label>
     </template>
     <script>
     export default {
       inheritAttrs: false
     }
     </script>
     ```
  
  4. 多个根节点的attribute如果没有显示的绑定，那么会报警告，我们必须手动的指定要绑定到哪一个属性上(由于vue3取消div根元素包裹)

  ```html
  <template :class="$attrs.class">伞兵一号</template>
  <template>伞兵2号</template>
  <template>伞兵3号</template>
  ```

## 子组件传递给父组件

### defineEmits

* 使用情景
  * 当子组件有一些事件发生的时候，比如在组件中发生了点击，父组件需要切换内容
  * 子组件有一些内容想要传递给父组件的时候

* 流程
  * 我们需要在子组件中定义好在某些情况下触发的事件名称
  * 和`defineProps`一样,使用泛型可以得到很好的代码提示
  
  ```html
  <h1 @click="butFn" class="green">{{ msg }}</h1>
  ...
  <script setup lang="ts">
  //子组件中给定绑定的名称
  interface testEmits {
    (e: 'em', data: number): void
    (e: 'ws', data: string): void
  }
  const emits = defineEmits<testEmits>()
  const butFn = () => {
    emits('em', 1)
  }
  </script>
  <!-- 父组件 -->
  <HelloWorld @em="pageFn" msg="You did it!" />
  <script setup lang="ts">
    const pageFn = (val: number) => {
      console.log(val)
    }
  </script>
  ```

### defineExpose

>在\<script setup>中使用的组件默认是关闭setup函数中的`retrun`,向外暴露属性的,不过可以使用`defineExpose`来暴露子组件的属性

* 问题:如果使用ts,父组件中的ref不能自动推断出子组件暴露的类型

```vue
<!-- 子组件 -->
<script setup lang="ts">
const testExpose = ref({ name: 'zhagnsan' })
const fn = () => {
  console.log('testExpose')
}
defineExpose({ testExpose, fn })
</script>
<!-- 父组件 -->
<script setup lang="ts">
const testE = ref()
const hello = () => {
  console.log(testE.value.testExpose.name)
  testE.value.fn()
}
</script>
...
<template>
 <div class="wrapper" @click="hello">
    <HelloWorld ref="testE" @em="pageFn" msg="You did it!" />
  </div>
</template>
```

## provide和inject

> 用于深层嵌套的组件,子组件想获取父组件甚至父父的部分内容

* 父组件有一个 `provide` 选项来提供数据(强烈建议写成函数的形式)
  * <span style="color:red">写成对象模式,这里的this并不会绑定到vm身上,而是一个未定义</sapn>
* 子组件有一个 `inject`选项来开始使用这些数据
* 组件层级`App.vue->Home.vue->HomeContent.vue`

```js
//App组件
  data() {
    return {
      names: ["a", "d", "v"],
    }
  },
  provide() {
    return {
      name: "zhangsan",
      age: 18,
      names:this.names.length
      //names:computed(()=>this.names.length)响应式的数据
    }
  }

  //HomeContent组件
  inject:["name","age","names"]
  //names.value拿取数据
```

* **关于响应式数据的处理**

* <span style="color:red">修改了`this.names`的内容，使用length的子组件会并不会不会是响应式的</span>
* provide中的内容并不是响应式的,而是及时通信,修改内容后并不会引起子组件反应
* 解决:
  * 使用vue3中的`computed`函数,他返回的是一个ref对象,需要用value取出

## 全局事件总线mitt

* Vue3从实例中移除了 `$on`、`$off` 和 `$once` 方法
* 安装mitt库:`npm install mitt`

* 在utils中封装一个`eventbus.js`工具

```js
import mitt from 'mitt'
const emitter=mitt()
export default emitter
```

* 监听与取消监听

```js
//Header组件中传入数据
  methods: {
    btnClick() {
      console.log("点击Header")
      emitter.emit("why",{name:"why",age:18})
    },
  },
//Footer组件中接收数据
  methods: {
    infomation(info) {
      console.log(info);
    }
  },
  created() {
    emitter.on("why", this.infomation);
    emitter.on("*", (type, info) => {
      console.log(type, info)
    })
  },
  //取消单独事件监听
  unmounted() {
    emitter.off("why", this.infomation)
  },
}
```

* 是通配符打印所有事件,type是接收类型即("why"),info是传入信息
* 取消所有监听`emitter.all.clear`
