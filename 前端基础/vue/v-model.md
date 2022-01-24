### 实现原理

#### 1. 作用在普通表单元素上
```
动态绑定了input的value指向了message变量，并且在触发input事件的时候去动态的把message设置为目标值。
```
```js
<input v-model='something>
```
等同于
```js
<input v-bind:value='message' v-on:input='mesage=$event.target.value'>
```
```
$event 当前事件对象
$event 当前事件对象dom
$event 当前dom的值
在@input方法中，value-->something
在:value中，something-->value
```
#### 2.作用在组件上
```
在自定义组件中，v-model默认会利用名为value的prop和名为input的事件
本质上是一个父子组件通信的语法糖，通过prop跟$emit来实现 因此父组件v-model语法糖本质上可以修改为
```
```js
<child :value="text" @input='function(e){text=e}'></child>
```
```
在组件的实现中，我们是可以通过v-model来配置子组件接收的prop属性以及派发的事件名称
```
```js
// 父组件
<parent-input v-model="parent" ></parent-input>
// 等价于
<parent-input :value="parent" @input="parent=$event.target.value"></parent-input>

// 子组件
 <input v-model="message" @input="onmessage"></>
 props:{
   value:message
 },
 methods:{
   onmessage(e){
     this.$emit('input',e.target.value)
   }
 }
```
