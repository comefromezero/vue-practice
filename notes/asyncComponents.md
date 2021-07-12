# 异步组件加载

应用场景：按需加载，加快首屏加载时间。

异步组件加载方式有三种：工厂函数加载、、import()加载、高级异步组件加载。

1. 工厂函数加载

``` Javascript
Vue.component('HelloWorld',function(resolve,reject){     　　//重写HelloWorld组件的定义
    require(['./components/HelloWorld'],function(res){       //利用webpack来打包，如果是载入别的组件，只需要修改组件名，以及组件路径即可，其它完全不变。
        resolve(res)
    })
})   //全局注册方式
```

一开始我一直在纠结了这个resolve和reject到底从哪里来，这个不必纠结，它们会在Vue内部由Vue自己处理好，用户所需要做的就是按照上面的形式将组件载入即可。

如果要用来载入别的组件，只需要修改组件名以及组件路径即可。

2. import()加载

``` Javascript
import Vue from 'vue'
import App from './App.vue'
import router from './router'

Vue.config.productionTip = false

Vue.component('HelloWorld',()=>import('./components/HelloWorld'))

new Vue({
  router,
  render: h => h(App)
}).$mount('#app')

//还可以局部注册

new Vue({
  // ...
  components: {
    'my-component': () => import('./my-async-component')   //局部注册组件，就像平时局部注册组件一样，这里只不过使用的是()=>import()来代替。
  }
})
```

如果我们想利用webpack将两个异步加载的组件打包成为一个文件，我们给import()中加上这样一个注释即可， /* webpackChunkName: "async" */，然后就变成了这样：

import( /* webpackChunkName: "async" */ './my-async-component'), import( /* webpackChunkName: "async" */'./components/HelloWorld')。


3. 高级异步组件加载

所谓高级异步组件加载只不过是让用户自定义很多异步组件加载的字段值，而上述两种方式的加载方式中，Vue自己设置了默认的字段值。

尽量用前面两种，这种还是少用，毕竟要设置的东西多，搞不好就出问题。

``` Javascript
new Vue({
  // ...
    components: {
        'my-component': () => {
            return {
                 // 需要加载的组件 (应该是一个 `Promise` 对象)
                component: import('./MyComponent.vue'),
                // 异步组件加载时使用的组件
                loading: LoadingComponent,
                // 加载失败时使用的组件
                error: ErrorComponent,
                // 展示加载时组件的延时时间。默认值是 200 (毫秒)
                delay: 200,
                // 如果提供了超时时间且组件加载也超时了，
                // 则使用加载失败时使用的组件。默认值是：`Infinity`
                timeout: 3000
            }
        }   //局部注册组件，就像平时局部注册组件一样，这里只不过使用的是()=>import()来代替。
    }
})
```