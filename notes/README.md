# 学习Vue的笔记

## vue基础之组件

* 非单文件组件

1. 创建一个组件，通过Vue.extend()来创建。

``` Javascript
const profile = Vue.extend({
  template: '<p>{{firstName}} {{lastName}} aka {{alias}}</p>', //如果是多行，就必须使用模板字符串了。
  data: function () {
    return {
      firstName: 'Walter',
      lastName: 'White',
      alias: 'Heisenberg'
    }
  }
}); //创建组件几个注意点：1）、组件中的data必须是一个函数；2）、组件没有el；
```
如果一个组件的data不是一个函数而是一个对象，那么该组件的所有实例的数据都将一样，从而引起实例间的数据污染。

2. 注册组件：全局注册or局部注册

``` Javascript
Vue.component('profile',profile);   //全局注册Profile组件，第一个参数是组件名，第二个参数是组件。

//全局注册的组件可以被所有Vue实例使用。
//组件名最好全小写并添加连字符。
```

``` Javascript
//局部注册组件：在对应的vue实例中通过components字段来包含需要使用的组件，被包含的组件可以被对应的实例中使用。
const vm = Vue({
    el:'#app',
    data:{},
    components:{
        'profile':profile,
    }, //这样在vm这个vue实例中就可以使用profile这个组件了。
})
```

3. 使用组件

``` Javascript

// 一般来说我们使用如下的方式（方式一）来使用组件:
//1）、使用组件需要实例化一个Vue对象，即let xxxx = Vue({...})。
//2）、实例化一个vue对象之后，将这个实例挂载到对应dom节点上。挂载的方式有两种：一种是在实例化的时候通过el这个选项来指明挂载的dom节点ID(例如#app)；另一种是在实例化一个Vue对象之后
//（假设是vm），在通过调用vm.mount('#app')将这个vm实例挂载到到对应的dom节点上。

// 方式一
const vm = Vue({
    el:'#app',
    data:{},
    components:{
        'profile':profile,
    }
})

// 方式二
const vm = Vue({
    data:{},
    components:{
        'profile':profile,
    }
}).$mount('#app');

// 或者
const vm = Vue({
    data:{},
    components:{
        'profile':profile,
    }
})
vm.$mount("#app");
```

``` Javascript
//一般来说我们使用上面这种方式，当然也可以直接实例化对应的组件，然后将这个实例挂载到对应的dom节点上。
//一般来说没人直接实例化组件，因为不好管理。
new profile({
    el:'#app'
});
//或者
new profile().$mount('#app');
```

* 单文件组件

注意点：一个单文件组件还不是一个真正意义上的vue组件，它仅仅包含了一个真正组件的全部的配置数据，它还需要经过vue-cli的编译之后才能成为真正的vue组件。

1. 定义一个单文件组件
1）、创建一个test.vue文件
2）、在test.vue文件中写入如下内容：
```
<template>
    <div id="itany">
        <h1>welcome</h1>
        <h2 @click="change">{{name}}</h2>
    </div>
</template>

<script>
    export default {
        name: "app",
        data(){
            return {
                name:'tom'
            }
        },
        methods:{
            change(){
                this.name='汤姆';
            }
        },
    }
</script>

<style scoped>
    body{
        background-color:#ccc;
    }
</style>

```

这个文件包含三部分结构:\<template\>、\<script\>、\<stype scoped\>。

此时，我们就定好了一个单文件组件。


2. 在别的文件中引入，然后通过components包含该组件，这样就能在组件中或者实例中使用该单文件组件了。

我们创建一个App.vue:

```
<template>
    <div><test></test></div>
</template>
<script>
    import test from ./components/test
    export default {
        components:{
            "test":test,
        },
    }
</script>
<style>

</style>
```
3. 然后我们在一个index.html中使用App.vue
``` html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>helloworld</title>
    <script src="./js/Vue.js"></script>
</head>
<body>
    <div id="root">
        <App></App>
    </div> 
</body>
<script>
    import App from ./App
    let vm = new Vue({
        el:"#root",
        components:{
            "App":App,
        }
    });
</script>

</html>
```
4. 使用Vue-cli编译


## Vue组件之间的通信

vue支持组件包含另外一个组件，例如A组件中包含了B组件，B组件包含了C组件和D组件，我们称A组件是B组件的父组件，B组件是A组件的子组件。B和C、B和D都是父子关系，C和D是兄弟关系，A和C是隔代关系。

### 父组件向子组件传值

通过props来传递数据。

### 通过$emit产生一个自定义事件来通知父组件

通过$emit产生一个自定义事件，然后在父组件中的模板中给子组件的标签上绑定一个该事件的处理函数，这样就能通过该事件的处理获取到相应的信息了。

更多的，我们不仅可以通过这种方式来通知父组件，还能通知祖先组件。

### 任意组件之间的通信

通过$emit产生自定义事件，通过$on来绑定该事件及处理函数。

### 注意点

组件之间的通信虽然也叫自定义事件，然而它的实现并不是通过事件流实现的，它实际上是通过内部实现的一套发布者订阅者模式来实现的通信。

* $emit产生的事件是广播机制，处于同一个Vue实例中的所有订阅者（通过执行vm.$on()订阅）都会收到该事件，然后执行相应的处理函数。

[具体使用请点击这里](https://segmentfault.com/a/1190000019208626)


## diff算法的几个注意点

同层比较，逐层查找，查找相同节点。

节点相同，则比较相同节点的下一层的子节点。

重复上述过程。

复用新旧相同节点，创建全新节点，删除旧节点。

[具体参考一](https://mp.weixin.qq.com/s?__biz=MzUxNjQ1NjMwNw==&mid=2247484442&idx=1&sn=26143f93c79390947920a8dc7b84bd14&chksm=f9a66e06ced1e71009a1aacfa134348f3b1e6a2e95a982b09fa2ad5559791650d84aad5eaeb9&token=655489829&lang=zh_CN&scene=21#wechat_redirect)

[具体参考二](https://mp.weixin.qq.com/s?__biz=MzUxNjQ1NjMwNw==&mid=2247484449&idx=1&sn=7f346b97a177218cc09fc50562ed121c&chksm=f9a66e3dced1e72b8a88fd0d78b5a5b8bd2e0ec95552e675d44923d368bba2ec438c520cd7be&cur_album_id=1619085427984957440&scene=189#rd)

## key来表示不同节点，防止vue复用该节点

diff算法利用key和tag名称来确定新旧节点是否是同一个节点，从而来确定是否复用该节点。

当我们不设置key的时候，key默认为undefined；如果新旧dom树种存在相同名称的节点，而key恰好都没设置的话，那么它们是相同的节点，则复用。

一般的节点复用不会产生什么问题，但是当输入框、选择框等复用的时候，可能会连同框中的内容一同复用，出现意向不到的效果，所以如果出现这种情况的话，

添加key就能告诉vue，不要复用这样的节点。

当然，某些时候，我们也需要利用vue这样的复用特性，例如有多个登录模式，可以复用用户输入的信息。



