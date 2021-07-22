# 深入响应式原理


Vue的核心原理就是ES5增加的Object.defineProperty方法修改一个Object的property的getter和setter这两个attributes来实现。当我们获取一个property的值的时候，就会执行与这个property同名的getter，当我们给一个property赋值的时候，就会执行与这个property同名的setter。那么Vue利用这个特性，重写一个property的setter和getter，在这两个函数中做了大量的操作，例如执行watch和执行重新渲染等操作，从而实现了数据更改，视图更新。

附：![响应式原理图](https://github.com/comefromezero/vue-practice/blob/main/notes/img/ReactivityInDepth.png)

## Vue将数据转换为响应式的过程<span id="1"><span>

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

## watcher的类型<span id="2"></span>

1. renderWatcher，Vue双向绑定的数据类型就是这种watcher,而这种watcher可能通知依赖多个property，与computed类似,例如：v-bind、模板{{}}等如果出现在同一视图中，那么

2. watch中定义的watcher。

3. computed中定义的watcher，这一类watcher属于lazy watcher，只有当其所依赖的property发生变化的时候，才会重新计算。正好对应了computed属性的缓存。


## 异步更新队列

    可能你还没有注意到，Vue 在更新 DOM 时是异步执行的。只要侦听到数据变化，Vue 将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更。
    如果同一个 watcher 被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和 DOM 操作是非常重要的。然后，在
    下一个的事件循环“tick”中，Vue 刷新队列并执行实际 (已去重的) 工作。Vue 在内部对异步队列尝试使用原生的 Promise.then、MutationObserver 
    和 setImmediate，如果执行环境不支持，则会采用 setTimeout(fn, 0) 代替。


上面是Vue官网对Vue究竟是如何响应数据变化的原理说明，比较简单浅显，下面通过源码来深刻分析Vue是如何侦测数据变化，然后又是在何时根据数据来更新视图的？

在一开始的的[Vue将数据转换为响应式的过程](#1)中我们说到当数据更新的时候，就会触发该property的set，在set方法中会直接通知订阅该property的所有watcher，下面我们来看一下该过程。

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
      dep.notify()                     //通知所有订阅的watcher数据已经更新。
    }
  })
}
```
可以看到，实际上set中是调用dep.notify来通知所有定于该property的watcher，我们来看notify：

``` Javascript
    /* 通知所有Watcher对象更新视图 */
    Dep.prototype.notify = function notify() {
        // stabilize the subscriber list first
        var subs = this.subs.slice();
        for (var i = 0, l = subs.length; i < l; i++) {   //通过循环来遍历每一个watcher
            //更新数据
            subs[i].update();    //调用对应的watcher的update方法。
        }
    }
```

直接看update方法：

``` Javascript
 Watcher.prototype.update = function update() {

        /* istanbul ignore else  */
        if (this.lazy) { //懒惰的 computed属性相关分支
            this.dirty = true;

        } else if (this.sync) { //如果是同步  watch中设置了sync选项的分支。

            //更新数据
            this.run();
        } else {

            //如果是多个观察者
            queueWatcher(this); //队列中的观察者   一般都是走这一条分支
        }
    };
```

在update中通过queueWatcher将一个watcher放入一个队列中，正如开头Vue文档中所说的，只要侦测到数据变化，Vue就会开启一个队列，将watcher放入队列之中。

我们继续看queueWatcher,看看它究竟是怎么来处理一个watcher放入队列的：

``` Javascript
function queueWatcher(watcher) {
        var id = watcher.id;     //获取wacher的id，该id是全局唯一的。
        if (has[id] == null) {  //判断该wacher是否已经存在于该队列中，如果已经存在，则不再将该watcher推入队列中，正好对应文档中说的，Vue将缓冲同一Tcik周期内的数据变化，
            has[id] = true;     //并只将同一watcher推入队列中一次。
            // flushing=true; //源码中就是将这个注释掉的，要注意。
            console.log(flushing)

            if (!flushing) {//fluashing标志，表示队列是否正处于执行watcher的状态之中，处于执行watcher的状态之中将禁止再往其中直接加入新的watcher，可以理解为一把锁。
                queue.push(watcher); //把观察者添加到队列中
            } else { //处于执行watcher状态中，新的wacher走此分支来加入到队列中。
                // if already flushing, splice the watcher based on its id 
                // if already past its id, it will be run next immediately. 
                var i = queue.length - 1;
                while (i > index && queue[i].id > watcher.id) {//寻找到该watcher的索引位置。
                    i--;
                }
                //根据id大小拼接插入在数组的哪个位置
                queue.splice(i + 1, 0, watcher);
            }
            console.log(waiting)

            // queue the flush
            if (!waiting) { //waiting标志表示队列的异步任务尚未开始排队，如果还没开始排队，则在第一个watcher加入到队列中之时,
                waiting = true; //直接开始在时间循环队列中排队，等待执行队列中所有wacher。
                //为callbacks 收集队列cb 函数 并且根据 pending 状态是否要触发callbacks 队列函数
                nextTick(
                    flushSchedulerQueue//该函数是一个回调，功能是执行队列中的所有watcher。
                );
            }
            //在本次事件循环中，第一次在队列中加入watcher之时，队列执行的回调就会去事件循环队列中排队，以获得更高的执行优先级。
            //队列执行回调开始排队之后，依然可以加入新的watcher。
        }
    }

```

在queueWatcher中，通过watcher.id判断该watchet是否已经加入过队列，如果没有，则将该watcher按照已经的办法加入到队列中。并且该函数在第一次加入watcher的时候，就调用nextTick将执行该队列的回调函数放入到事件循环队列中排队。那么nextTick是如何将一个回调函数放入到事件循环队列中呢？

``` Javascript
function nextTick(cb, ctx) {
        //cb 回调函数
        //ctx this的指向
        var _resolve;
        //添加一个回调函数到队列里面去

        callbacks.push(function () {   //通过一个匿名函数将参数中的回调函数直接传入到callbacks队列中，callbacks有一个对应的函数叫flushcallbacks，
            if (cb) {                  //该函数会执行callbacks中所有的函数。
                //如果cb存在 并且是一个函数就执行
                try {
                    cb.call(ctx);
                } catch (e) {
                    //如果不是函数则报错
                    handleError(e, ctx, 'nextTick');
                }
            } else if (_resolve) {
                //_resolve 如果存在则执行
                _resolve(ctx);
            }
        });
        console.log('==callbacks==')
        console.log(callbacks)
        console.log(pending)

        if (!pending) {

            pending = true;
            //下面两个分支中的两个函数都会将flushcallbacks函数作为回调函数放入到事件循环队列中。
            if (useMacroTask) {

                macroTimerFunc(); //传统异步任务setTimeout触发排队。
            } else {
                microTimerFunc(); //ES6新加入的promise异步任务触发microtask队列排队。
            }
        }


        // $flow-disable-line
        if (!cb && typeof Promise !== 'undefined') {
            //如果回调函数不存在 则声明一个Promise 函数
            return new Promise(function (resolve) {
                _resolve = resolve;
            })
        }
    }
```

在nextTick中，其通过一个队列将回到回调函数缓存起来，并通过一个异步任务函数将该队列相关的一个函数flushcallbacks推入到事件循环队列中，其并不是真正的将回调直接加入到事件循环的队列中。

    另外，nextTick其实是一个全局性的函数，在文档中也有说明，可以通过调用nextTick保证回调函数在下一次事件周期执行该回调。

等到当前的call stack为空的时候，事件循环就会执行事件循环队列中的所有回调函数了，当然也包括我们通过nextTick放入的flushcallbacks，然后我们真正的回调就能通过callbacks队列得到执行。


下面再来看看flushShelderQueue是如何执行watcher的：

``` Javascript
/**
     * Flush both queues and run the watchers. 刷新两个队列并运行监视程序。
     * 更新观察者 运行观察者watcher.run() 函数 并且   调用组件更新和激活的钩子
     */
    function flushSchedulerQueue() {
        flushing = true;
        var watcher, id;

        // Sort queue before flush.
        // This ensures that:
        // 1. Components are updated from parent to child. (because parent is always
        //    created before the child)
        // 2. A component's user watchers are run before its render watcher (because
        //    user watchers are created before the render watcher)
        // 3. If a component is destroyed during a parent component's watcher run,
        //    its watchers can be skipped.
        //刷新前对队列排序。
        //这确保:
        // 1。组件从父组件更新到子组件。因为父母总是在孩子之前创建)
        // 2。组件的用户观察者在其呈现观察者之前运行(因为用户观察者是在渲染观察者之前创建的)
        // 3。如果一个组件在父组件的监视程序运行期间被销毁，可以跳过它的观察者。
        //观察者根据id去排序
        queue.sort(function (a, b) {
            return a.id - b.id;
        });  //这里对watcher按照id进行排序，目的是为了让renderwatcher优先运行。

        // do not cache length because more watchers might be pushed 不要缓存长度，因为可能会推入更多的观察者
        // as we run existing watchers 我们运行现有的观察者
        for (index = 0; index < queue.length; index++) {
            watcher = queue[index]; //获取单个观察者
            id = watcher.id;
            has[id] = null;
            watcher.run(); //运行watcher
            // in dev build, check and stop circular updates. 在dev build中，检查并停止循环更新。
            if ("development" !== 'production' && has[id] != null) {
                circular[id] = (circular[id] || 0) + 1;
                if (circular[id] > MAX_UPDATE_COUNT) {
                    warn(
                        'You may have an infinite update loop ' + (
                            watcher.user
                                ? ("in watcher with expression \"" + (watcher.expression) + "\"")
                                : "in a component render function."
                        ),
                        watcher.vm
                    );
                    break
                }
            }
        }

        // keep copies of post queues before resetting state 在重置状态之前保留post队列的副本
        var activatedQueue = activatedChildren.slice(); // 浅拷贝
        var updatedQueue = queue.slice();// 浅拷贝

        //清空观察者watcher队列中的数据
        resetSchedulerState();

        // call component updated and activated hooks 调用组件更新和激活的钩子
        callActivatedHooks(activatedQueue);  //
        callUpdatedHooks(updatedQueue);

        // devtool hook
        /* istanbul ignore if */
        //触发父层flush 钩子函数
        if (devtools && config.devtools) {
            devtools.emit('flush');
        }
    }
```

在flushSchedulerQueue中，对队列中的watcher按照id从小到大进行排队，然后依次执行watcher，对于renderwatcher来说就是执行其render，对于computed watcher来说就是执行其回调函数，对于watch中用户定义的watcher来说也是执行对应的回调函数；执行完所有的watcher之后，清空本次watcher队列，然后调用Actived生命周期钩子，接着调用Updated生命周期钩子。随后该函数执行完毕，意味着本次的更新已经完毕。

### 总结

当数据更新就会触发该property的set，在set中会通知所有的订阅该property的watcher，通过notify调用watcher的update方法，在update方法中调用queueWatcher将对应的watcher放入到队列中，并于第一次放入watcher的时候调用nextTick函数将该队列的执行函数flushSchedulerQueue放入到全局的回调队列中，然后在nextTick中会将本轮的回调队列执行函数放入到事件循环的队列之中。当事件循环调度开始之后，全局的回调函数都得到执行，flushSchedulerQueue也会得到执行，从而令Watcher得到执行。这其中包括了renderwatcher，然后视图就会使用新的数据就行渲染了。

### nextTick的原理

首先假设完全已经了解并掌握了浏览器的[事件循环调度机制](https://blog.csdn.net/lengye7/article/details/118771503)。

在Vue中使用两种方案：

第一种：macroTimeFunc,该方案会将一个任务放入到任务队列中，该任务会在UI渲染完成之后得到事件循环调度，然后得以执行。在不同的浏览器平台中，macorTimeFunc采用3种降级方案：优先使用setImmediate，次之messagechannel，最后使用setTimeout。

第二种：microTimeFunc，该方案将一个任务放入到微任务队列中，该任务会在当前调用栈为空的时候得到事件循环的调度，然后得以执行，其在UI渲染之前完成。在不同的浏览器平台中，microTimeFunc采用2种方案：优先使用promise，如果浏览器不支持promise,则将microTimeFunc降级为macroTimeFunc。

上面两种方案中，默认情况下是使用第二种方案的，如果浏览器不支持使用第二种方案，则会使用第一种方案。

nextTick实际上就是利用了事件循环的原理，通过使用一些API将需要执行的任务放入到队列中（根据不同的浏览器使用第一种或者第二种方案），从而实现在当前任务执行完（执行完，调用栈为空）之后，投递到队列中的任务得到运行（在UI渲染之前（第一种），或者在UI渲染之后（第二种））。

在这两种方案中，自然是第二种要好，因为第二种的任务能够在渲染之前就更新数据，那么在本次渲染中就能够更新视图。如果是第一种方案，任务会在渲染之后更新数据，那么就只能在下一次渲染中更新视图了。

而在这里面有一个细节问题：dom节点只要通过dom相关的API操作添加到了html文档中，我们就可以对dom节点使用相关API进行操作了，它并不需要等到UI渲染之后才可以被访问，所以无论是使用第一种还是第二种方案，我们可以给nextTick中传入一个操作dom节点的回调。

### renderwatcher

前面我们提到有三种不同的watcher，对于computedwatcher和watch定义的watcher来说，毫无疑问是在initComputed和initWatch中初始化并定义的。那么renderwatcher是在哪里定义的呢？

实际上，renderwatcher是在mount函数中定义的。因此按照顺序来说，这三种watcher的定义顺序分别是computedwatcher、watch中定义的watcher、renderwatcher，那么它们的id大小也是按照定义的先后顺序从小到大排列的。

接下来我们来看看renderwatcher究竟是怎么样被定义的？

我们知道Vue函数调用_init函数进行初始化，在_init函数的最后通过vm.$mount函数将实例挂载到一个dom节点上，然后实例会得到渲染。

直接看源码：(需要注意，Vue有两个版本，一个Vue.js是不带模板编译器的版本(该版本模板在打包的时候就已经编译了)，另外一个是带有模板编译器的版本，会在加载的时候编译模板。为了清除Vue的实际渲染过程，我们这里分析的是带模板编译器的版本。)

``` Javascript
// public mount method 安装方法 实例方法挂载 vm
    // 手动地挂载一个未挂载的实例。

    Vue.prototype.$mount = function (el,  //真实dom 或者是string
                                     hydrating  //新的虚拟dom vonde
    ) {
         debugger
        el = el && inBrowser ? query(el) : undefined; //获取到需要挂载的dom节点
        return mountComponent(  //真正mount的地方。
                                this,
                                el,
                                hydrating
                             )
    };
        var mount = Vue.prototype.$mount; //缓存上一次的Vue.prototype.$mount

    // Vue 的$mount()为手动挂载，
    // 在项目中可用于延时挂载（例如在挂载之前要进行一些其他操作、判断等），之后要手动挂载上。
    // new Vue时，el和$mount并没有本质上的不同。

    Vue.prototype.$mount = function (el, hydrating) { //重写Vue.prototype.$mount
                                                      //_init中实际上执行的是这里这个$mount方法，然后在
        el = el && query(el); //获取dom
        /* istanbul ignore if */
        //如果el 是body 或者文档 则警告
        if (el === document.body || el === document.documentElement) {
            "development" !== 'production' && warn(
                "Do not mount Vue to <html> or <body> - mount to normal elements instead."
            );
            return this
        }
        //获取参数
        var options = this.$options;
        // resolve template/el and convert to render function
        //解析模板/el并转换为render函数
        if (!options.render) {
            //获取模板字符串
            var template = options.template;

            if (template) { //如果有模板

                if (typeof template === 'string') { //模板是字符串

                    //模板第一个字符串为# 则判断该字符串为 dom的id
                    if (template.charAt(0) === '#') {
                        console.log(template)

                        template = idToTemplate(template); //获取字符串模板的innerHtml
                        console.log(template)

                        /* istanbul ignore if */
                        if ("development" !== 'production' && !template) {
                            warn(
                                ("Template element not found or is empty: " + (options.template)),
                                this
                            );
                        }
                    }
                } else if (template.nodeType) { //如果template 是don节点 则获取他的html
                    template = template.innerHTML;
                } else {
                    //如果什么都是不是则发出警告
                    {
                        warn('invalid template option:' + template, this);
                    }
                    return this

                }
            } else if (el) {

                //如果模板没有，dom节点存在则获取dom节点中的html 给模板
                template = getOuterHTML(el);
                console.log(template)

            }
            if (template) {
                /* istanbul ignore if */
                //监听性能监测
                if ("development" !== 'production' && config.performance && mark) {
                    mark('compile');
                }
                //创建模板
                console.log('==options.comments==')
                console.log(options.comments)

                var ref = compileToFunctions(  //编译模板
                    template, //模板字符串
                    {
                        shouldDecodeNewlines: shouldDecodeNewlines, //flase //IE在属性值中编码换行，而其他浏览器则不会
                        shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref, //true chrome在a[href]中编码内容
                        delimiters: options.delimiters, //改变纯文本插入分隔符。修改指令的书写风格，比如默认是{{mgs}}  delimiters: ['${', '}']之后变成这样 ${mgs}
                        comments: options.comments //当设为 true 时，将会保留且渲染模板中的 HTML 注释。默认行为是舍弃它们。
                    },
                    this
                );
                // res.render = createFunction(compiled.render, fnGenErrors);
                //获取编译函数 是将字符串转化成真正js的函数
                console.log('==ref.render==')
                console.log(ref.render)
                console.log(ref)
                console.log('==ref.render-end==')
                // res.render = createFunction(compiled.render, fnGenErrors);
                // //字符串转化js 创建一个集合函数
                // res.staticRenderFns = compiled.staticRenderFns.map(function (code) {
                //     return createFunction(code, fnGenErrors)
                // });



                // ast: ast, //ast 模板
                //render: code.render, //code 虚拟dom需要渲染的参数函数
                //staticRenderFns: code.staticRenderFns  //空数组

                //这样赋值可以有效地 防止 引用按地址引用，造成数据修改而其他对象也修改问题，
                var render = ref.render;
                var staticRenderFns = ref.staticRenderFns;

              /*
               render 是  虚拟dom，需要执行的编译函数 类似于这样的函数
               (function anonymous( ) {
                    with(this){return _c('div',{attrs:{"id":"app"}},[_c('input',{directives:[{name:"info",rawName:"v-info"},{name:"data",rawName:"v-data"}],attrs:{"type":"text"}}),_v(" "),_m(0)])}
                 })
               */
                options.render = render; //将render函数放入实例的render中。
                options.staticRenderFns = staticRenderFns;
                console.log(options);
                console.log(options.render);

                /* istanbul ignore if */
                if ("development" !== 'production' && config.performance && mark) {
                    mark('compile end');
                    measure(("vue " + (this._name) + " compile"), 'compile', 'compile end');
                }
            }
        }
        console.log(render)
        console.log(el)
        console.log(hydrating)


        //执行$mount方法 一共执行了两次 第一次是在9000多行那一个  用$mount的方法把扩展挂载到dom上
        return mount.call(
                            this,
                            el, //真实的dom
                            hydrating //undefined
                )
    };
```

上面的执行过程：实际在_init中先执行第二个$mount方法，然后在第二个$mount方法中调用了第一个$mount方法。在执行的过程中，编译模板获得render函数，然后将render函数放入到当前实例中，最后挂载渲染。

继续看mountComponent:

``` Javascript
    function mountComponent(
                                vm,  //vnode
                                el,  //dom
                                hydrating
                             ) {
        vm.$el = el; //dom
        console.log(vm.$options.render)

        //如果参数中没有渲染
        if (!vm.$options.render) { //实例化vm的渲染函数，虚拟dom调用参数的渲染函数
            //创建一个空的组件
            vm.$options.render = createEmptyVNode;

            {
                /* istanbul ignore if */
                //如果参数中的模板第一个不为# 号则会 警告
                if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
                    vm.$options.el || el) {
                    warn(
                        'You are using the runtime-only build of Vue where the template ' +
                        'compiler is not available. Either pre-compile the templates into ' +
                        'render functions, or use the compiler-included build.',
                        vm
                    );
                } else {
                    warn(
                        'Failed to mount component: template or render function not defined.',
                        vm
                    );
                }
            }
        }



        //执行生命周期函数 beforeMount
        callHook(vm, 'beforeMount');
        //更新组件
        var updateComponent;
        /* istanbul ignore if */
        //如果开发环境
        if ("development" !== 'production' && config.performance && mark) {
            updateComponent = function () {
                var name = vm._name;
                var id = vm._uid;
                var startTag = "vue-perf-start:" + id;
                var endTag = "vue-perf-end:" + id;

                mark(startTag); //插入一个名称 并且记录插入名称的时间
                var vnode = vm._render();
                mark(endTag);
                measure(("vue " + name + " render"), startTag, endTag);

                mark(startTag); //浏览器 性能时间戳监听
                //更新组件
                vm._update(vnode, hydrating);
                mark(endTag);
                measure(("vue " + name + " patch"), startTag, endTag);
            };
        } else {
            updateComponent = function () {
                console.log(vm._render())


                //直接更新view试图
                vm._update(

                    /*
                     render 是  虚拟dom，需要执行的编译函数 类似于这样的函数
                     (function anonymous( ) {
                     with(this){return _c('div',{attrs:{"id":"app"}},[_c('input',{directives:[{name:"info",rawName:"v-info"},{name:"data",rawName:"v-data"}],attrs:{"type":"text"}}),_v(" "),_m(0)])}
                     })
                     */
                    vm._render(), //先执行_render,返回vnode
                    hydrating
                );
            };
        }

        // we set this to vm._watcher inside the watcher's constructor
        // since the watcher's initial patch may call $forceUpdate (e.g. inside child
        // component's mounted hook), which relies on vm._watcher being already defined
        //我们将其设置为vm。在观察者的构造函数中
        //因为观察者的初始补丁可能调用$forceUpdate(例如inside child)
        //组件的挂载钩子)，它依赖于vm。_watcher已经定义
        //创建观察者
        new Watcher(
            vm,  //vm vode
            updateComponent, //数据绑定完之后回调该函数。更新组件函数 更新 view试图 ，这里以此函数作为观察对象传入Wather中。
            noop, //回调函数
            null, //参数
            true //表示这是一个renderwatcher。
            /* isRenderWatcher */);
        hydrating = false;

        // manually mounted instance, call mounted on self
        // mounted is called for render-created child components in its inserted hook
        //手动挂载实例，调用挂载在self上
        // 在插入的钩子中为呈现器创建的子组件调用// mount
        if (vm.$vnode == null) {
            vm._isMounted = true;
            //执行生命周期函数mounted
            callHook(vm, 'mounted');
        }

        return vm
    }
```

在上面的函数中，最后调用Watcher，并传入updateComponent作为观察对象，以及将isRenderWatcher设置为true（表示这是一个renderWatcher），这两个参数正是关键所在。到这里，我们实际上已经获得了一个renderwatcher了。最后，会调用mounted生命周期钩子，挂载渲染完成。

我们来看Watcher中，它们是怎么工作的？

``` Javascript
    /**
     * A watcher parses an expression, collects dependencies,
     * and fires callback when the expression value changes.
     * This is used for both the $watch() api and directives.
     * *观察者分析表达式，收集依赖项，
     *并在表达式值更改时触发回调。
     *这用于$watch() api和指令。
     * 当前vue实例、updateComponent函数、空函数。
     */
    var Watcher = function Watcher(
                                   vm, //vm dom
                                   expOrFn,  //获取值的函数，或者是更新viwe试图函数
                                   cb, //回调函数,回调值给回调函数
                                   options, //参数
                                   isRenderWatcher//是否渲染过得观察者
    ) {
        console.log('====Watcher====')
        this.vm = vm;
        //是否是已经渲染过得观察者
        if (isRenderWatcher) { //把当前 Watcher 对象赋值给 vm._watcher上
            vm._watcher = this;
        }
        //把观察者添加到队列里面 当前Watcher添加到vue实例上
        vm._watchers.push(this);
        // options
        if (options) { //如果有参数
            this.deep = !!options.deep; //深层监听，与Vue文档的deep选项相同。
            this.user = !!options.user; //这个选项为true表示该watcher是用户在watch选项中或者vm.$watch中定义的。
            this.lazy = !!options.lazy; //该选项为true表示此watcher是computed watcher。
            this.sync = !!options.sync; //对应于Vue文档中的immediate属性。
        } else {

            this.deep = this.user = this.lazy = this.sync = false;
        }
        this.cb = cb; //回调函数
        this.id = ++uid$1; // uid for batching uid为批处理  监听者id
        this.active = true; //激活
        this.dirty = this.lazy; // for lazy watchers 对于懒惰的观察者
        this.deps = [];    // 观察者队列
        this.newDeps = []; // 新的观察者队列
        // 内容不可重复的数组对象
        this.depIds = new _Set();
        this.newDepIds = new _Set();
        // 把函数变成字符串形式
        this.expression = expOrFn.toString();
        // parse expression for getter
        //getter的解析表达式
        if (typeof expOrFn === 'function') {
            //获取值的函数
            this.getter = expOrFn;
        } else {
            //如果是keepAlive 组件则会走这里
            //path 因该是路由地址
            if (bailRE.test(path)) {  //  匹配上 返回 true     var bailRE = /[^\w.$]/;  //匹配不是 数字字母下划线 $符号   开头的为true
                return
            }

            // //匹配不上  path在已点分割
            // var segments = path.split('.');
            // return function (obj) {
            //
            //     for (var i = 0; i < segments.length; i++) {
            //         //如果有参数则返回真
            //         if (!obj) {
            //             return
            //         }
            //         //将对象中的一个key值 赋值给该对象 相当于 segments 以点拆分的数组做obj 的key
            //         obj = obj[segments[i]];
            //     }
            //     //否则返回一个对象
            //     return obj
            // }

            //匹配不是 数字字母下划线 $符号   开头的为true

            this.getter = parsePath(expOrFn);
            if (!this.getter) { //如果不存在 则给一个空的数组
                this.getter = function () {
                };
                "development" !== 'production' && warn(
                    "Failed watching path: \"" + expOrFn + "\" " +
                    'Watcher only accepts simple dot-delimited paths. ' +
                    'For full control, use a function instead.',
                    vm
                );
            }
        }
        this.value = this.lazy ?
            undefined :
            this.get();
    };
```

在Watcher中，第二个参数正是updateComponent,它是一个函数，这样该函数就会进入如下分支：

``` Javascript
        if (typeof expOrFn === 'function') {
            //获取值的函数
            this.getter = expOrFn;
        }
```

该watcher的getter的正好是ipdateComponent。在Watcher函数的最后，会执行this.get()函数，在get()函数中，会调用this.getter，即相当于执行了updateComponent函数，然后该renderwatcher对应的Vue实例得到渲染。

这正是一个Vue实例初始化之后渲染的关键所在。初始化一个renderwatcher，在renderwatcher初始化完成之后渲染该Vue实例。

上面就是Vue实例初始化并挂载渲染的过程。初始化完成的之后，该Vue实例会获得一个对应的renderwatcher。

### renderwatcher是如何更新视图的？

当一个property的set被触发，相应的watcher就会得到执行，renderwatcher也在其中。

前面我们已经知道watcher最后通过run方法得到运行。

``` Javascript
    Watcher.prototype.run = function run() {
        if (this.active) { //活跃
            var value = this.get(); //获取值 函数 expOrFn
            if (
                value !== this.value ||  //如果值不相等
                // Deep watchers and watchers on Object/Arrays should fire even 深度观察和对象/数组上的观察应该是均匀的
                // when the value is the same, because the value may 当值相等时，因为值可以
                // have mutated. 有突变。
                isObject(value) || //或者值的object
                this.deep  //获取deep为true
            ) {
                // set new value
                var oldValue = this.value; //获取旧的值
                this.value = value; //新的值赋值
                if (this.user) { //如果是用户在watch选项中或者通过vm.$watch方法定义的就会走这条分支
                    try {
                        this.cb.call(this.vm, value, oldValue); //更新回调函数  获取到新的值 和旧的值
                    } catch (e) {
                        handleError(e, this.vm, ("callback for watcher \"" + (this.expression) + "\""));
                    }
                } else {
                    this.cb.call(this.vm, value, oldValue);//更新回调函数  获取到新的值 和旧的值
                }
            }
        }
    };
```
在run方法中，第一步就是运行this.get()函数，在之前我们就已经说过了，对于一个renderwatcher来说，这就相当于运行了updateComponent，而该函数正好用于一个Vue实例的渲染，于是更新数据驱动了视图的改变，视图得到更新（即真实的dom节点得到更新）。

当所有的微任务执行完毕之后，浏览器会进入到UI渲染阶段，由于检测到html文档有所更改，于是浏览器重新渲染整个html文档，于是更新后的视图被真正呈现给用户了。








