# 深入响应式原理


Vue的核心原理就是ES5增加的Object.defineProperty方法修改一个Object的property的getter和setter这两个attributes来实现。当我们获取一个property的值的时候，就会执行与这个property同名的getter，当我们给一个property赋值的时候，就会执行与这个property同名的setter。那么Vue利用这个特性，重写一个property的setter和getter，在这两个函数中做了大量的操作，例如执行watch和执行重新渲染等操作，从而实现了数据更改，视图更新。

附：![响应式原理图](https://github.com/comefromezero/vue-practice/blob/main/notes/img/ReactivityInDepth.png)

## Vue将数据转换为响应式的过程

Vue实例化的初始化过程中，在beforeCreate之后，就会真正进入初始化阶段，开始初始化props、data、methods、watch等。正是在这个初始化阶段，Vue会将data中的数据都转换为响应式
数据，而这个过程在_initData()函数中完成。

``` Javascript
<!-源码目录：src/instance/internal/state.js-->
Vue.prototype._initData = function () {
    var dataFn = this.$options.data
    var data = this._data = dataFn ? dataFn() : {}
    if (!isPlainObject(data)) {
      data = {}
      process.env.NODE_ENV !== 'production' && warn(
        'data functions should return an object.',
        this
      )
    }
    var props = this._props
    // proxy data on instance
    var keys = Object.keys(data)
    var i, key
    i = keys.length
    while (i--) {
      key = keys[i]
      // there are two scenarios where we can proxy a data key:
      // 1. it's not already defined as a prop
      // 2. it's provided via a instantiation option AND there are no
      //    template prop present
      if (!props || !hasOwn(props, key)) {
        this._proxy(key)  //这一步会将data中的数据都代理到当前实例中，这就是我们能通过this.xxxx直接访问数据的原因。当然，我们也能通过this.data.xxxx访问数据。
      } else if (process.env.NODE_ENV !== 'production') {
        warn(
          'Data field "' + key + '" is already defined ' +
          'as a prop. To provide default value for a prop, use the "default" ' +
          'prop option; if you want to pass prop values to an instantiation ' +
          'call, use the "propsData" option.',
          this
        )
      }
    }
    // observe data
    observe(data, this)   //这一步是真正的核心，遍历data对象中的所有property，递归的处理各个property，给其添加响应式。
  }
```

这个函数这种，真正核心的就两个地方，一个是_proxy,另外一个就是observe。

我们先来看_proxy，这个函数将data中的数据代理到当前实例上，使我们能够通过this.xxxx访问data中的数据。

``` Javascript
<!-源码目录：src/instance/internal/state.js-->
Vue.prototype._proxy = function (key) {
    if (!isReserved(key)) {
      // need to store ref to self here
      // because these getter/setters might
      // be called by child scopes via
      // prototype inheritance.
      var self = this
      Object.defineProperty(self, key, {   //这里我们可以看到Vue也是通过Object.defineProperty来实现代理的，其在当前实例中定义了同名的key的set和get函数。
        configurable: true,                //在函数内部，其访问_data中的同名的key的property。
        enumerable: true,                  //通过这种方式，从而实现了this.xxxx的访问。
        get: function proxyGetter () {     //当我们通过this.xxxx来访问的时候，直接触发get，然后get给我们返回了this._data.xxxx的值，等于实际上我们访问了this.xxxx。
          return self._data[key]           //当我们通过this.xxxx=12345设置一个值的时候，直接触发set，然后set直接又设置了this._data.xxxx=12345，等于实际上我们     
        },                                 //设置了this.xxxx的值。
        set: function proxySetter (val) {
          self._data[key] = val
        }
      })
    }
  }

```

接着我们再看observ，这才是重头戏。

``` Javascript
<!-源码目录：src/observer/index.js-->
export function observe (value, vm) {
  if (!value || typeof value !== 'object') {  //简单的来说只要不是对象直接返回，这里之所以要加一个!value，主要是为了处理typeof null的结果为object的问题。
    return
  }
  var ob
  if (
    hasOwn(value, '__ob__') &&
    value.__ob__ instanceof Observer         //如果一个对象不具备__ob__这个不可被遍历到property，那么说明其还没有被observe过，那就会进入下面的分支，进行
  ) {                                        //Overser(value),显然，当_data首次被observe的时候，必然会进入这一条分支。
    ob = value.__ob__
  } else if (
    shouldConvert &&
    (isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (ob && vm) {
    ob.addVm(vm)
  }
  return ob
}
```

这个函数中真正核心的就是Observer了，_data首次进入这个函数，就会进入到Observer的分支。我们来看Observer:

``` Javascript
<!-源码目录：src/observer/index.js-->
export function Observer (value) {
  this.value = value
  this.dep = new Dep()
  def(value, '__ob__', this)
  if (isArray(value)) {          //如果是数组则走这里，进行特殊的处理。
    var augment = hasProto
      ? protoAugment
      : copyAugment
    augment(value, arrayMethods, arrayKeys)
    this.observeArray(value)
  } else {
    this.walk(value)           //如果是对象，则走这里，遍历对象的每一个property，然后处理。
  }
}

```

当_data首次进入的时候，必然会走this.walk这条分支，我们先来分析这一条分支。

``` Javascript
<!-源码目录：src/observer/index.js-->
Observer.prototype.walk = function (obj) {   //正如函数名字所言，把Object中的每一个property走了一遍。
  var keys = Object.keys(obj)
  for (var i = 0, l = keys.length; i < l; i++) {
    this.convert(keys[i], obj[keys[i]])
  }
}
```

这里调用convert转换没一个property。我们来看convert:

``` Javascript
<!-源码目录：src/observer/index.js-->
Observer.prototype.convert = function (key, val) {
  defineReactive(this.value, key, val)   //实际上是在调用这个函数
}

```

想必,defineReactive应该是真正对每一个property做处理的地方：

``` Javascript
<!-源码目录：src/observer/index.js-->
export function defineReactive (obj, key, val) {
  var dep = new Dep()
  var property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }
  // cater for pre-defined getter/setters
  var getter = property && property.get
  var setter = property && property.set
  var childOb = observe(val)     //这个函数我们之前说过，如果不是对象，直接返回，所以在这里，只有当这个值是对象的时候才会继续走下去。也就是说，如果是对象就递归调用
  Object.defineProperty(obj, key, { //observe。
    enumerable: true,
    configurable: true,                                  //看到这里，我们就找到Vue真正为data中一个property做响应式处理的地方，在这里，给每个property设置get和set。
    get: function reactiveGetter () {                    //对应的函数各自做了很多事情，它们除了返回值和设置值之外，还做了其他额外的事情。
      var value = getter ? getter.call(obj) : val        //get中收集了定于这个property的订阅者。
      if (Dep.target) {                                  //set中执行了notify通知这个property的订阅者。
        dep.depend()                                    // Dep这个数据结构就是用来收集watcher的，每个watcher在初始化的时候，都会访问对应的peoperty，所以会触发
        if (childOb) {                                  // get，从而触发get对watch的收集过程。
          childOb.dep.depend()                          //具体的收集过程可以查看initWatch函数，需要注意的是：watcher分为两类，一类是我们watch和computed中定义的，
        }                                               //另一类是我们通过其他方式双向绑定的renderWatcher。核心就是Watcher这个类。
        if (isArray(value)) {                           //到这里，我们基本弄清楚了Vue的响应式原理了。get/set触发之后会直接执行watcher，watcher负责更新视图，
          for (var e, i = 0, l = value.length; i < l; i++) {   //响应完成，正好对应了原理图中的data到watcher的箭头，箭头到re-render的箭头。
            e = value[i]                                       //wathcer那边也是很绕的，慎看。
            e && e.__ob__ && e.__ob__.dep.depend()
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val
      if (newVal === value) {
        return
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = observe(newVal)
      dep.notify()
    }
  })
}

```

上面我们分析了对象的转换过程，首先data是一个对象，它被observe,然后调用oberseve递归的observe它的property。

当一个property的值是一个原始值的时候，直接设置set/get。

当一个property的值是一个对象的时候，递归Observe。当这个递归返回之后，为这个property设置set/get。（记住：property保存的是对象（数组也是对象）的引用值。这个引用值不会在对象变动或者数组变动的时候改变，所以vue不能通过这个property的set/get监测到对应的变化。）

数组也是一个对象，但是数组与普通对象相比，会有另外特殊的处理。

我们接下来看Vue对property是数组的话，Vue到底是怎么处理的：

首先我们应当明确一件事，Vue最开始遍历data中property，在调用Object.defienProperty之前都会Observe一次这个property，意思是对一个对象继续递归observe。

数组和Object都是对象，所以都会在这个函数中走下去，对于数组来说，最终走到Observer的数组分支：

``` Javascript
<!-源码目录：src/observer/index.js-->
export function Observer (value) {
  this.value = value
  this.dep = new Dep()
  def(value, '__ob__', this)
  if (isArray(value)) {          //如果是数组则走这里，进行特殊的处理。
    var augment = hasProto
      ? protoAugment
      : copyAugment
    augment(value, arrayMethods, arrayKeys)
    this.observeArray(value)
  } else {
    this.walk(value)           //如果是对象，则走这里，遍历对象的每一个property，然后处理。
  }
}
```

数组分支中有两个重要的处理函数，第一个是augment，第二个是ovserveArray。

先来看augment，它是一个动态绑定的函数，我们只看protoAugment，一般来说一个数组肯定具备__proto__属性:

``` Javascript
    function protoAugment(target, src, keys) {
        /* eslint-disable no-proto */
        target.__proto__ = src;
        /* eslint-enable no-proto */
    }
//这个函数很简单，就是将target的__proto__设置为src，在augment的调用中，它实际在做的事情是：将一个实例数组的__proto__设置为arrayMethods对象。
```

接下来看arrayMethods的处理：

这是一段全局代码。

``` Javascript
var arrayProto = Array.prototype;
var arrayMethods = Object.create(arrayProto);  //从Array.prototype创建一个新的对象实例，这个对象具备Array.prototype的所有property，同时它的__proto__是Object。
var methodsToPatch = [                         //对这7中数组方法进行了处理，从而当我们使用这7中数组方法对数组进行操作的时候，Vue能监测到数组的变化。
    'push',
    'pop',
    'shift',
    'unshift',
    'splice',
    'sort',
    'reverse'
];
methodsToPatch.forEach(function (method) {
    console.log('methodsToPatch')
    // cache original method
    var original = arrayProto[method];
    console.log('==method==')
    console.log(method)
    console.log('==original==')
    console.log(original)
    def(arrayMethods, method, function mutator() {   //在这里，为arrayMethods对象重写这7种方法，将push、pop...等都设置为mutator这个函数。
        console.log('==def_original==')              //当我们调用push等方法的时候，实际上在调用mutator这个函数，我们来看看它实际在做什么。
        console.log(original)
        var args = [], len = arguments.length;
        while (len--) args[len] = arguments[len];
        var result = original.apply(this, args);     //首先调用原数组方法对数组进行更改。
        var ob = this.__ob__;
        console.log('this.__ob__')
        console.log(this.__ob__)

        var inserted;
        switch (method) {
            case 'push':
            case 'unshift':
                inserted = args;
                break
            case 'splice':
                inserted = args.slice(2);
                break
        }
        if (inserted) {
            //观察数组数据
            ob.observeArray(inserted);            //由于加入了新数据，为数组新加入的每一个数据都建立响应式，如果新加入的数据过大，就比较影响性能了。
        }                                        
           // notify change
         //更新通知
        ob.dep.notify();                          //然后通知watcher数组更新了，没错，这是最重要的一步，通知watcher。
        console.log('====result====')
           console.log(result)
         return result
    });
});
```

Vue通过augment对实例数组的push等方法进行了重写，重写的方法除了具备原方法的功能之外，还额外做了通知watcher的事情，从而当我们通过这些方法对数组进行操作的时候，watcher得到通知notify，Vue能监测到数组的变化，一个比较hack的做法。

我们再看observeArray:

``` Javascript
 Observer.prototype.observeArray = function observeArray(items) {   //这里的处理很简单，直接为Observe每个数组元素。
        for (var i = 0, l = items.length; i < l; i++) {             //又回到之前的地方了，如果数组元素值是原始值，直接返回了。
            console.log('items[i]')                                 //如果是对象，就递归遍历为每个property设置set/get。
            console.log(items[i])                                   //如果是数组，就按照数组的处理办法处理。
                                                                    //可以知道的是：Vue并没有为每个数组元素设置set/get。

            observe(items[i]);
        }
    };
```

### 总结

Observe这个函数对于传入的值的处理：

如果是原始值，直接返回（递归返回条件）;如果是对象，则遍历该对象的property，Observe这个property之后(递归调用)，就为这个property设置set/get;如果是数组，则走数组的特殊处理后，Observe该数组的每个元素（递归调用），并不会为数组的每个元素都设置set/get，所以这也就是为啥Vue无法通过索引来访问数组元素触发响应式了。这个函数最后完成的功能就是，将对象的每一个property都设置一个set/get，数组本身会有特殊处理但不为数组元素设置set/get。

这里是真的挺绕的，它这个递归是放在多个不同的函数之中完成的，乍看之下，根本发现不了，难受哟。

一开始，Vue以data为参数调用Observe，由于data是一个对象，所以observe遍历data，Observe data的所有property，Observe返回之后为这些property都设置set/get。这是一个递归调用过程。

在Observe每一个property的过程中，如果遇到了原始值，自然直接返回，然后就为其设置set/get。

如果遇到了Object（array另说），就走Object的分支，递归Observe每个Object的property，最终遇到原始值，递归返回，为property设置set/get。在这个递归过程，将对象的property都设置了set/get。

如果遇到了数组，数组本身做特殊处理，然后Observe数组的每个元素(这又回到递归了)，递归返回之后并不为每个数组元素都设置set/get。

总之，经过这个过程，无论data中的对象和数组嵌套有多深，最终每个property都会被设置set/get，data中的每个数组也都得到了特殊处理,从而实现了data中数据的响应式。


## watcher和dep的关系

1. Vue在初始化data中的每个数据的时候，都会在defineReactive中为每个property new一个dep，也就是说一个property对应一个dep，这个dep中保存的是本property的订阅者watcher。
2. 当一个watcher通过Watcher函数实例化的时候，都会执行其所依赖的那个data中的property的getter，然后触发这个property的get，get中做了两件事儿，第一件事儿就是将对应的dep放入当前这个watcher的dep收集队列中，第二件事就是将当前这个watcher放入到本property对应的dep中。
3. 一个watcher可能有多个依赖的property，例如依赖一个对象，然后设置deep选项，那么这个watcher就会依赖对象中的每一个property，然后这个watcher的dep队列中就会保存多个property的dep。
4. 当在某个地方更新了data中的某个property，就会触发它的set，set中做了两件事儿，第一件事就是更改值，第二件事儿就是挨个通知本dep中的所有watcher。收到通知之后，watcher就会被放入一个队列中，等待nextTick执行。
5. 一个watcher每执行一次就会动态更新自己的property依赖列表（渲染wathcer的依赖property可能会有改变，computed和watch中定义的一定不会改变，所以这个处理应该是为了渲染wathcer），如果不再依赖某个property，则会取消订阅该property，将本watcher从该property的dep列表中删除。
6. watcher会保存依赖的property的dep，而property的dep中保存着订阅（依赖）本property的watcher。

## watcher的类型

1. renderWatcher，Vue双向绑定的数据的watcher就会是这种类型的watcher。

2. watch中定义的watcher。

3. computed中定义的watcher，这一类watcher属于lazy watcher，只有当其所依赖的property发生变化的时候，才会重新计算。正好对应了computed属性的缓存。


## 异步更新队列

    可能你还没有注意到，Vue 在更新 DOM 时是异步执行的。只要侦听到数据变化，Vue 将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更。
    如果同一个 watcher 被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和 DOM 操作是非常重要的。然后，在
    下一个的事件循环“tick”中，Vue 刷新队列并执行实际 (已去重的) 工作。Vue 在内部对异步队列尝试使用原生的 Promise.then、MutationObserver 
    和 setImmediate，如果执行环境不支持，则会采用 setTimeout(fn, 0) 代替。


