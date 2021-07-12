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

## 组件的一些简写形式<span id="1">
data(){} = data: function(){}

components:{组件名} = components:{"组件标签名"，组件名} ，其中前一种方式会直接使用组件名作为组件标签名。



## 组件全局注册和局部注册的优缺点

Vue实例关于组件的组件：当一个Vue实例被加载的时候，在其局部注册和全局注册所有组件要被加载到该实例中。

全局注册的组件可以在注册之后，在其他任意地方使用，这是其优点，避免了每次需要使用一个组件都需要先局部注册一下，这是全局注册的优点。但是由于全局注册的组件在任何Vue实例加载的时候，全局组件都会被加载一次，可能我们在某个实例中不需要使用这些全局组件，这就造成了额外的性能开销，造成了浪费。

局部注册的组件在我们使用的时候，必须在使用的地方通过components来声明注册一下，然后才能使用，我们可以按需注册，不需要的组件可以注册，避免了额外的开销。但是，每次都需要注册一下，当在我们在大量地方需要使用一些相同的组件的时候，给我们带来了一些重复工作。

总结：折中考虑，对于一些公共组件可以使用全局注册，一些非公共组件使用局部注册。

## Vue组件实例与Vue实例

当一个Vue实例加载的时候，其内部使用的Vue组件也都会创建实例，每使用一次组件，就会创建一个组件实例。

通过对VueComponents函数的分析，可以看到，VueComponents实际上继承了Vue，我们知道Vue实例是通过Vue()构造函数创建的，而Vue组件实际上就是一个Vuecomponent()对象，我们创建一个组件实例，实际上就是在创建一个Vuecomponent实例。

因此Vue实例和组件实例实际上都有一个共同的原型，那就是Vue对象。


## Vue组件之间的通信

vue支持组件包含另外一个组件，例如A组件中包含了B组件，B组件包含了C组件和D组件，我们称A组件是B组件的父组件，B组件是A组件的子组件。B和C、B和D都是父子关系，C和D是兄弟关系，A和C是隔代关系。

### 父组件向子组件传值

通过props来传递数据。props引入的是父组件设置在子组件标签上的属性,在props中通过属性名来引入。

### 子组件通过$emit产生一个自定义事件来通知父组件

子组件通过$emit产生一个自定义事件，然后在父组件中的模板中给子组件的标签上绑定一个该事件的处理函数，这样就能通过该事件的处理获取到相应的信息了。

更多的，我们不仅可以通过这种方式来通知父组件，还能通知祖先组件。

### 任意组件之间的通信

通过$emit产生自定义事件，通过$on来绑定该事件及处理函数。

### 注意点

组件之间的通信虽然也叫自定义事件，然而它的实现并不是通过事件流实现的，它实际上是通过内部实现的一套发布者订阅者模式来实现的通信。

* $emit产生的事件是广播机制，处于同一个Vue实例中的所有订阅者（通过执行vm.$on()订阅）都会收到该事件，然后执行相应的处理函数。

[具体使用请点击这里](https://segmentfault.com/a/1190000019208626)

[实现原理请请点击这里](https://juejin.cn/post/6844904115806404615)

## 组件中的一些选项

### name

允许组件模板递归的调用自身。

给组件命名，会产生更加友好的警告信息。

通过Vue.component()全局注册的组件会自动使用id作为name的值。

### data()

组件中data选项必须是一个函数。

### props

用来接收父组件的数据,访问父组件的v-bind绑定的属性或者原生属性。要想在props中引入一个数据对象，就必须用v-bind将这个数据对象与一个属性绑定，这个属性可以是自己乱写的名字都可以。总结，props可以引入父作用域写在该组件上的属性，如果该属性通过v-bind绑定到了一个数据对象上，则props中引入的该属性是响应式的；如果该属性不是通过v-bind绑定到一个数据对象上，那么props中引入的该属性的值就一直等于定义时的值。

### inheritAttrs

默认情况下父作用域的不被认作 props 的 attribute 绑定 (attribute bindings) 将会“回退”且作为普通的 HTML attribute 应用在子组件的根元素上。

通过设置 inheritAttrs 到 false，这些默认行为将会被去掉。

### 单文件组件中style scope

使用scoped修饰的style表示这些style只在组件内部使用。

### vm.$attrs

包含了父作用域中不作为 prop 被识别 (且获取) 的 attribute 绑定 (class 和 style 除外)。当一个组件没有声明任何 prop 时，这里会包含所有父作用域的绑定 (class 和 style 除外)，并且可以通过 v-bind="$attrs" 传入内部组件——在创建高级别的组件时非常有用。

如果一个v-bind绑定的属性的数据通过子组件的props被访问，那么该属性名不会被正确渲染。

实际上，经过测试，一个属性如果没有通过v-bind绑定，只要能够被渲染到dom节点上(也就是说原生的alt=1000也是能够被子组件访问到的)其也可以通过this.$attrs访问到。

在组件中通过this.$attrs访问。

### vm.$listeners

包含了父作用域中的 (不含 .native 修饰器的) v-on 事件监听器。它可以通过 v-on="$listeners" 传入内部组件——在创建更高层次的组件时非常有用。

在组件中通过this.$listeners访问。

### vm.$props

在组件中通过this.$props可以访问组件中的props对象。

### vm.$data

在组件中通过this.$data可以访问组件中的data对象。

### vm.$root

通过this.$root直接访问根实例对象，然后我们就能通过this.$root.xxxx来访问跟实例中的其他property了。

### this

在vue中，vue会自动的将this绑定到当前实例上，在组件中使用this需要注意的是，this会指向所属的那个组件的对应实例。我们可以直接通过this.xxxx来访问组件中的property。但是需要注意的是，不要在watch，computed，methods中使用箭头函数作为一个回调或者方法，因为这些选项中箭头函数的this跟组件定义的环境上下文相关。但是我们可以在一个一个function中使用箭头函数，因为该function的this被设置绑定到了当前实例，该function中的箭头函数的this与该function一致。

如果我们在生命周期钩子中使用了异步操作，或者在实例中的函数方法里使用了异步操作，一定要记得使用箭头函数来作为异步操作的回调函数，因为箭头函数中的this与定义它的环境上下文的this一致，如果在钩子中定义了箭头函数，那么箭头函数的this跟钩子的this一样（钩子的this被Vue设置为当前实例），如果在方法中定义了箭头函数，那么箭头函数的this跟方法的this一样（显然方法的this被Vue设置为当前实例）。

记住，如果希望在异步回调中通过this操作当前实例的数据，一定使用箭头函数作为回调函数。


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


## 简写形式

[组件中的简写形式](#1)

v-on简写形式@

v-bind简写形式:

methods:{
    //直接在methods里写方法
    test1(){}
}相当于

methods:{
    test1:function(){}
}

computed:{
    test(){

    }
}
//相当于
computed:{
    test:function(){}
}

## Vue对标签属性的处理

1. 我们可以通过像原生一样直接在标签上指定一个属性名并设置值。
2. 我们还可以通过v-bind指令将一个属性与data中的某个property绑定起来，从而实现属性值的响应式。
3. 我们可以在组件的标签名上使用上述方式，也可以在原生标签上使用上述方式。如果我们在一个组件的自定义标签名上使用属性，那么经过渲染之后，这些属性会出现在该组件的最外层的那个dom标签上。
4. 需要注意的是：如果一个属性被子组件通过props引入了，那么该属性不会被渲染到子组件的根节点上，不管该属性是v-bind绑定的还是像原生一样设置的。
5. 子组件可以通过this.$attrs来访问父作用域中设置的属性(不能访问到props中的属性)。
6. 子组件通过v-bind="$attrs"继续传入内部组件。(现在想想，elementUI应该就是使用这种方式来传递属性的吧，哈哈哈。)

## Vue中对事件的处理

1. 我们可以通过v-on指令在一个标签上或者一个组件上设置一个事件处理函数。
2. vue还定义了一些与event相关的修饰器对应到event的原生方法：
    * .stop - 调用 event.stopPropagation()，阻止冒泡。
    * .prevent - 调用 event.preventDefault(),取消该事件(如果可行的话)。
    * .capture - 添加事件侦听器时使用 capture 模式,在捕获阶段该事件处理函数被调用。
    * .self - 只当事件是从侦听器绑定的元素本身触发时才触发回调。
    * .{keyCode | keyAlias} - 只当事件是从特定键触发时才触发回调。
    * .native - 监听组件根元素的原生事件。
    * .once - 只触发一次回调。
    * .left - (2.2.0) 只当点击鼠标左键时触发。
    * .right - (2.2.0) 只当点击鼠标右键时触发。
    * .middle - (2.2.0) 只当点击鼠标中键时触发。
    * .passive - (2.3.0) 以 { passive: true } 模式添加侦听器

3. Vue关于event对象的处理，如果是v-on:click = doSomething的形式，那么event对象将被自动传入;如果是v-on:click = doSomething()的形式，那么必须改成这样v-on:click=doSomething($event)，显式传入$event对象。doSomething的形式应该为：doSomething(event){ 在内部就可以通过event来使用event对象了}。

4. 这些处理函数可以在methods中进行定义。

5. Vue中并没有提供删除事件绑定的办法，如果我们有这种需求，可以自己实现一个自定义指令。

6. 子组件可以通过this.$listener来获取到父作用域下的v-on(除了.native修饰符之外的)事件监听器，并可以通过v-on="$listeners"继续传入内部组件。

## Vue中的自定义事件

Vue提供了四个方法来操作自定义事件，需要注意的是，自定义事件并不是event，而是Vue自己实现的一套发布者订阅者模型。

四个方法：vm.$on( event, callback )、vm.$once( event, callback )、vm.$off( [event, callback] )、vm.$emit( eventName, […args] )。

* vm.$emit

该方法抛出一个自定义事件，在组件中通过this.$emit来调用。

* vm.$on

该方法用来绑定一个自定义事件，在组件中通过this.$on来调用。

* vm.$once

该方法也是用来绑定一个自定义事件，与vm.$on所不同的是，其绑定的处理函数只会执行一次，在组件中通过this.$once来调用。

* vm.$off

该方法用来移除一自定义事件的监听处理器,在组件中通过this.$off来调用。


## 常用的一些指令

v-show、v-if、v-else、v-else-if、v-for、v-on、v-bind、v-model、v-model。

### v-show与v-if的区别？

v-show通过设置display属性来显示和隐藏一个内容。

v-if通过删除dom节点以及重新渲染来隐藏和显示一个内容。

### v-model

v-model用于input，根据input类型的不同而行为不同，默认情况下，其监听input事件，通过修饰符.lazy可以切换到change事件。其最常用的场景就是绑定input的value。


## computed 和 watch的对比

* computed

    * computed声明的属性是响应式的。
    * computed中的属性的依赖属性可以有多个，其中任意一个发生变化都会触发该属性重新计算。但是需要注意的是，如果该依赖属性是组件之外的属性，那么当该属性变化的时候，computed不会重新计算。
    * computed中的属性是缓存的，计算完成之后如果依赖属性没有发生变化，则不再重新计算。缓存是computed最重要的特性。
    * computed中声明的属性可以在本组件中其他地方使用。
    * 默认情况下，computed只会有getter，但是可以显示的为其创建一个setter，该setter接受一个参数，该参数为computed属性的新设置的值。
    * 不要在computed中使用箭头函数作为回调，因为computed中的箭头函数中的this会绑定到组件定义的环境上下文中（一般情况下为全局上下文）。
    * computed中声明的属性对应的function会在实例加载的时候执行一次，因此不要在function里做超出计算该属性值之外的任何操作，这应该也是Vue不推荐在computed里做复杂计算和异步操作已经API调用了吧。

* watch
    * watch一个回调函数对应一个被观察的属性，被观察的属性必须是本实例中的属性。
    * 如果有多个属性需要被观察，那么需要给每个被观察的属性都写上一个回调。
    * watch的回调函数接受两个参数，第一个参数是被观察属性的新值，第二个参数是被观察属性的旧值。
    * watch中的function在实例加载的时候不会被执行，这应该也是其更加通用的地方，可以做任何操作，而不仅仅只是计算某个属性值，可以做异步操作等。

总结：多个属性变动的变化引发的简单计算可以使用computed，不推荐在computed中使用异步操作；观察单个属性，需要使用异步以及大量复杂计算操作等使用watch。

## data、computed、props、methods

Vue实例或者组件实例在加载的过程中，会将这四个选项中的property全家加载为实例的property，我们可以通过this.propertyName来访问这些property。

data、computed、props中的数据都是响应式的。

## 生命周期钩子

### 注意点

所有的生命周期钩子自动绑定 this 上下文到实例中，因此你可以访问数据，对 property 和方法进行运算。这意味着你不能使用箭头函数来定义一个生命周期方法 (例如 created: () => this.fetchTodos())。这是因为箭头函数绑定了父上下文，因此 this 与你期待的 Vue 实例不同，this.fetchTodos 的行为未定义。

钩子类型：beforeCreate、created、beforeMount、mounted、beforeUpdate、updated、beforeDestroy、destroyed、activated、deactivated、errorCaptured。

### beforeCreate

在实例初始化之后，数据观测 (data observer) 和 event/watcher 事件配置之前被调用。

此时我们还不能访问各种实例数据。

这个阶段适合添加一些loading事件，例如根据$route进行重定向操作。

### created

在实例创建完成后被立即调用。在这一步，实例已完成以下的配置：数据观测 (data observer)，property 和方法的运算，watch/event 事件回调。然而，挂载阶段还没开始，$el property 目前尚不可用。

此时各种数据都已经准备好了，但是还没有开始渲染，下阶段开始渲染。

这个阶段适合添加一些后台数据请求，更新data、methods等。

### beforeMount

在挂载开始之前被调用：相关的 render 函数首次被调用。

这个时候处于渲染之前。页面渲染要用到的所有数据必须在这之前赋值完成，否则会引起渲染不正常。

### mounted

实例被挂载后调用，这时 el 被新创建的 vm.$el 替换了。如果根实例挂载到了一个文档内的元素上，当 mounted 被调用时 vm.$el 也在文档内。

此时已经渲染完成，真实的dom节点已经被创建了。

需要注意的是：Vue并不能保证此时所有子组件也已经挂载渲染完成，因此如果想要保证子组件都已经渲染完成，即整个视图都已经渲染完成，Vue提供了vm.$nextTick来供我们使用。

这个阶段我们可以添加一些依赖于dom的操作，例如添加事件监听。

### beforeUpdate

数据更新时调用，发生在虚拟 DOM 打补丁之前。这里适合在更新之前访问现有的 DOM，比如手动移除已添加的事件监听器。

此时只发生了数据更新还没有发生重新渲染。

可以在这个阶段进一步修改data中的数据,或者做一些简单的数据过滤。

### updated

由于数据更改导致的虚拟 DOM 重新渲染和打补丁，在这之后会调用该钩子。

当这个钩子被调用时，组件 DOM 已经更新，所以你现在可以执行依赖于 DOM 的操作。然而在大多数情况下，你应该避免在此期间更改状态。如果要相应状态改变，通常最好使用计算属性或 watcher 取而代之。

此时，重新渲染已经完成了。

需要注意的是：updated 不会保证所有的子组件也都一起被重绘。如果你希望等到整个视图都重绘完毕，可以在 updated 里使用 vm.$nextTick。

这个阶段渲染完成，我们可以做一些依赖于dom的操作，例如添加dom事件监听。

### beforeDestroy

实例销毁之前调用。在这一步，实例仍然完全可用。我们仍然可以访问实例中的数据。

可以趁着实例销毁之前，我们保存一些持久化数据。

### destroyed

实例销毁后调用。该钩子被调用后，对应 Vue 实例的所有指令都被解绑，所有的事件监听器被移除，所有的子实例也都被销毁。

此时我们已经不能访问实例中的数据。

这个时候可以给用户提示已经关闭应用，哈哈哈哈。

### activated

被 keep-alive 缓存的组件激活时调用。

### deactivated

被 keep-alive 缓存的组件停用时调用。

### 附[!生命周期钩子图示](https://github.com/comefromezero/vue-practice/blob/main/notes/img/lifecycle.png)

## Vue实例的加载过程

[!picture](https://github.com/comefromezero/vue-practice/blob/main/notes/img/vueInitInstance.png)


## extends 与 mixins

这两个选项使用不多，个人认为也就用于需要大量复用代码的场景。

extends允许我们在一个组件中继承成一个组件的options，从而实现代码复用。

这个选项主要用于单文件组件中，当发现一个组件不能满足全部需求时，则可以选择继承其中一部分功能，然后在另外一个组件中编写扩展的功能。
``` Javascript
var CompA = { ... }

// 在没有调用 `Vue.extend` 时候继承 CompA
var CompB = {
  extends: CompA,
  ...
}
```

这里留下一个问题：extends继承的逻辑是什么？

mixins允许我们在一个组件中接收外部的选项对象，这些mixins对象最终会被合并到组件的options中，然后用于创建实例。
``` Javascript
var mixin = {
  created: function () { console.log(1) }
}
var vm = new Vue({
  created: function () { console.log(2) },
  mixins: [mixin]
})
// => 1
// => 2
```

这里再留下一个问题：mixins与本组件中的选项是如何合并的？

这里留下的两个问题是至关重要的，因为这涉及到使用这两个选项的安全问题，如果不熟悉它们的继承与合并逻辑，可能会写出超出意料之外的代码。
