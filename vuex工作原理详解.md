# vuex工作原理详解

> 引文从：https://www.jianshu.com/p/d95a7b8afa06



## 前言

vuex作为vue官方出品的状态管理框架，以及其简单API设计、便捷的开发工具支持，在中大型的vue项目中得到很好的应用。作为**flux**架构的后起之秀，吸收了前辈**redux**的各种优点，完美的结合了**vue**的**响应式**数据，个人认为开发体验已经超过了**React + Redux**这对基友。

在项目启动vue开发后的这几个月中，越发对vuex的原理感到好奇，今天将这几日的所学总结成文，希望能帮到对vuex好奇的童鞋们。

## 理解computed

使用vuex中store中的数据，基本上离不开vue中一个常用的属性computed。官方一个最简单的例子如下



```javascript
var vm = new Vue({
  el: '#example',
  data: {
    message: 'Hello'
  },
  computed: {
    // 计算属性的 getter
    reversedMessage: function () {
      // `this` 指向 vm 实例
      return this.message.split('').reverse().join()
    }
  }
})
```

不知大家有没有思考过，vue的computed是如何更新的，为什么当vm.message发生变化时，vm.reversedMessage也会自动发生变化？

我们来看看vue中data属性和computed相关的源代码。



```javascript
// src/core/instance/state.js
// 初始化组件的state
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  // 当组件存在data属性
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  // 当组件存在 computed属性
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

initState方法当组件实例化时会自动触发，该方法主要完成了初始化data,methods,props,computed,watch这些我们常用的属性，我们来看看我们需要关注的`initData`和`initComputed`（为了节省时间，去除了不太相关的代码）

先看看**initData这条线**



```javascript
// src/core/instance/state.js
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  // .....省略无关代码
  
  // 将vue的data传入observe方法
  observe(data, true /* asRootData */)
}

// src/core/observer/index.js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value)) {
    return
  }
  let ob: Observer | void
  // ...省略无关代码
  ob = new Observer(value)
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

在初始化的时候observe方法本质上是实例化了一个Observer对象，这个对象的类是这样的



```javascript
// src/core/observer/index.js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value
    // 关键代码 new Dep对象
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    // ...省略无关代码
    this.walk(value)
  }

  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      // 给data的所有属性调用defineReactive
      defineReactive(obj, keys[i], obj[keys[i]])
    }
  }
}
```

在对象的构造函数中，最后调用了**walk**方法，该方法即遍历data中的所有属性，并调用`defineReactive`方法，**defineReactive**方法是**vue**实现 MDV(Model-Driven-View)的基础，本质上就是代理了数据的set,get方法，当数据修改或获取的时候，能够感知（当然**vue**还要考虑数组，Object中嵌套Object等各种情况，本文不在分析）。我们具体看看`defineReactive`的源代码



```javascript
// src/core/observer/index.js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 重点，在给具体属性调用该方法时，都会为该属性生成唯一的dep对象
  const dep = new Dep()

  // 获取该属性的描述对象
  // 该方法会返回对象中某个属性的具体描述
  // api地址https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertyDescriptor
  const property = Object.getOwnPropertyDescriptor(obj, key)
  // 如果该描述不能被更改，直接返回，因为不能更改，那么就无法代理set和get方法，无法做到响应式
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set

  let childOb = !shallow && observe(val)
  // 重新定义data当中的属性，对get和set进行代理。
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      // 收集依赖， reversedMessage为什么会跟着message变化的原因
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
        }
        if (Array.isArray(value)) {
          dependArray(value)
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      // 通知依赖进行更新
      dep.notify()
    }
  })
}
```

我们可以看到，在`所代理的属性`的`get`方法中，当dep.Target存在的时候会调用`dep.depend()`方法，这个方法非常的简单，不过在说这个方法之前，我们要认识一个新的类`Dep`

Dep 是 vue 实现的一个处理依赖关系的对象，
 主要起到一个纽带的作用，就是连接 reactive data 与 watcher,代码非常的简单



```javascript
// src/core/observer/dep.js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      // 更新 watcher 的值，与 watcher.evaluate() 类似，
      // 但 update 是给依赖变化时使用的，包含对 watch 的处理
      subs[i].update()
    }
  }
}

// 当首次计算 computed 属性的值时，Dep 将会在计算期间对依赖进行收集
Dep.target = null
const targetStack = []

export function pushTarget (_target: Watcher) {
  // 在一次依赖收集期间，如果有其他依赖收集任务开始（比如：当前 computed 计算属性嵌套其他 computed 计算属性），
  // 那么将会把当前 target 暂存到 targetStack，先进行其他 target 的依赖收集，
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

export function popTarget () {
  // 当嵌套的依赖收集任务完成后，将 target 恢复为上一层的 Watcher，并继续做依赖收集
  Dep.target = targetStack.pop()
}
```

代码非常的简单，回到调用`dep.depend()`方法的时候，当`Dep.Target`存在，就会调用，而`depend方法`则是将该dep加入`watcher`的`newDeps`中,同时，将`所访问当前属性`的`dep`对象中的`subs`插入当前Dep.target的watcher.看起来有点绕，不过没关系，我们一会跟着例子讲解一下就清楚了。

讲完了代理的get,方法，我们讲一下代理的set方法，set方法的最后调用了`dep.notify()`,当设置data中具体属性值的时候，就会调用该属性下面的`dep.notify()`方法，通过`class Dep`了解到，notify方法即将加入该dep的watcher全部更新，也就是说，当你修改**data**中某个属性值时，会同时调用`dep.notify()`来更新依赖该值的所有`watcher`。

介绍完了`initData`这条线，我们继续来介绍`initComputed`这条线，这条线主要解决了什么时候去设置`Dep.target`的问题（如果没有设置该值，就不会调用**dep.depend()**, 即无法获取依赖）。



```javascript
// src/core/instance/state.js
const computedWatcherOptions = { lazy: true }
function initComputed (vm: Component, computed: Object) {
  // 初始化watchers列表
  const watchers = vm._computedWatchers = Object.create(null)
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (!isSSR) {
      // 关注点1，给所有属性生成自己的watcher, 可以在this._computedWatchers下看到
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    if (!(key in vm)) {
      // 关注点2
      defineComputed(vm, key, userDef)
    }
  }
}
```

在初始化computed时，有2个地方需要去关注

1. 对每一个属性都生成了一个属于自己的Watcher实例，并将 **{ lazy: true }**作为options传入
2. 对每一个属性调用了defineComputed方法(本质和data一样，代理了自己的set和get方法，我们重点关注代理的**get**方法)

我们看看**Watcher**的构造函数



```javascript
// src/core/observer/watcher.js
constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: Object
  ) {
    this.vm = vm
    vm._watchers.push(this)
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // 如果初始化lazy=true时（暗示是computed属性），那么dirty也是true,需要等待更新
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.getter = expOrFn // 在computed实例化时，将具体的属性值放入this.getter中
    // 省略不相关的代码
    this.value = this.lazy
      ? undefined
      : this.get()
  }
```

除了日常的初始化外，还有2行重要的代码

> this.dirty = this.lazy
>  this.getter = expOrFn

在**computed**生成的**watcher**，会将watcher的lazy设置为true,以减少计算量。因此，实例化时，this.dirty也是true,标明数据需要更新操作。我们先记住现在**computed中初始化对各个属性生成的watcher的dirty和lazy都设置为了true**。同时，将computed传入的属性值**（一般为funtion）**,放入**watcher**的**getter**中保存起来。

我们在来看看第二个关注点**defineComputed**所代理属性的get方法是什么



```javascript
// src/core/instance/state.js
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    // 如果找到了该属性的watcher
    if (watcher) {
      // 和上文对应，初始化时，该dirty为true,也就是说，当第一次访问computed中的属性的时候，会调用 watcher.evaluate()方法；
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

当`第一次`访问computed中的值时，会因为初始化`watcher.dirty = watcher.lazy`的原因，从而调用evalute()方法，evalute()方法很简单,就是调用了watcher实例中的**get**方法以及设置**dirty = false**,我们将这两个方法放在一起



```javascript
// src/core/instance/state.js
evaluate () {
  this.value = this.get()
  this.dirty = false
}
  
get () {  
// 重点1，将当前watcher放入Dep.target对象
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    // 重点2，当调用用户传入的方法时，会触发什么？
    value = this.getter.call(vm, vm)
  } catch (e) {
  } finally {
    popTarget()
    // 去除不相关代码
  }
  return value
}
```

在get方法中中，第一行就调用了**pushTarget**方法，其作用就是将**Dep.target**设置为所传入的watcher,即所访问的**computed**中属性的**watcher**,
 然后调用了`value = this.getter.call(vm, vm)`方法，想一想，调用这个方法会发生什么？

**this.getter** 在Watcher构建函数中提到，本质就是用户传入的方法，也就是说，**this.getter.call(vm, vm)**就会调用用户自己声明的方法，那么如果方法里面用到了 **this.data**中的值或者其他被用**defineReactive**包装过的对象，那么，访问this.data.或者其他被**defineReactive**包装过的属性，是不是就会访问被代理的该属性的get方法。我们在回头看看
 **get**方法是什么样子的。

> 注意:我讲了其他被用defineReactive，这个和后面的vuex有关系，我们后面在提



```csharp
get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      // 这个时候，有值了
      if (Dep.target) {
        // computed的watcher依赖了this.data的dep
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
        }
        if (Array.isArray(value)) {
          dependArray(value)
        }
      }
      return value
    }
```

代码注释已经写明了，就不在解释了，这个时候我们走完了一个依赖收集流程，知道了computed是如何知道依赖了谁。最后根据this.data所代理的**set**方法中调用的**notify**,就可以改变this.data的值，去更新所有依赖this.data值的computed属性value了。

那么，我们根据下面的代码，来简易拆解获取依赖并更新的过程



```jsx
var vm = new Vue({
  el: '#example',
  data: {
    message: 'Hello'
  },
  computed: {
    // 计算属性的 getter
    reversedMessage: function () {
      // `this` 指向 vm 实例
      return this.message.split('').reverse().join()
    }
  }
})
vm.reversedMessage // =>  olleH
vm.message = 'World' // 
vm.reversedMessage // =>  dlroW
```

1. 初始化 data和computed,分别代理其set以及get方法, 对data中的所有属性生成唯一的dep实例。
2. 对computed中的reversedMessage生成唯一watcher,并保存在vm._computedWatchers中
3. 访问 **reversedMessage**，设置Dep.target指向reversedMessage的watcher,调用该属性具体方法**reversedMessage**。
4. 方法中访问this.message，即会调用this.message代理的get方法，将this.message的**dep**加入reversedMessage的**watcher**,同时该dep中的**subs**添加这个**watcher**
5. 设置**vm.message = 'World'**，调用message代理的set方法触发**dep的notify**方法'
6. 因为是computed属性，只是将**watcher**中的**dirty**设置为true
7. 最后一步**vm.reversedMessage**，访问其get方法时，得知**reversedMessage**的**watcher.dirty**为true,调用**watcher.evaluate()**方法获取新的值。

这样，也可以解释了为什么有些时候当computed没有被访问（或者没有被模板依赖），当修改了this.data值后，通过vue-tools发现其**computed**中的值没有变化的原因，因为没有触发到其**get**方法。

## vuex插件

有了上文作为铺垫，我们就可以很轻松的来解释vuex的原理了。

我们知道，vuex仅仅是作为vue的一个插件而存在，不像Redux,MobX等库可以应用于所有框架，vuex只能使用在vue上，很大的程度是因为其高度依赖于vue的computed依赖检测系统以及其插件系统，

通过[官方文档](https://cn.vuejs.org/v2/guide/plugins.html)我们知道，每一个vue插件都需要有一个公开的install方法，vuex也不例外。其代码比较简单，调用了一下applyMixin方法，该方法主要作用就是在所有组件的**beforeCreate**生命周期注入了设置**this.$store**这样一个对象，因为比较简单，这里不再详细介绍代码了，大家自己读一读编能很容易理解。



```javascript
// src/store.js
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```



```javascript
// src/mixins.js
// 对应applyMixin方法
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])

  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```

我们在业务中使用vuex需要类似以下的写法



```javascript
const store = new Vuex.Store({
    state,
    mutations,
    actions,
    modules
});
```

那么 **Vuex.Store**到底是什么样的东西呢？我们先看看他的构造函数



```javascript
// src/store.js
constructor (options = {}) {
  const {
    plugins = [],
    strict = false
  } = options

  // store internal state
  this._committing = false
  this._actions = Object.create(null)
  this._actionSubscribers = []
  this._mutations = Object.create(null)
  this._wrappedGetters = Object.create(null)
  this._modules = new ModuleCollection(options)
  this._modulesNamespaceMap = Object.create(null)
  this._subscribers = []
  this._watcherVM = new Vue()

  const store = this
  const { dispatch, commit } = this
  this.dispatch = function boundDispatch (type, payload) {
    return dispatch.call(store, type, payload)
}
  this.commit = function boundCommit (type, payload, options) {
    return commit.call(store, type, payload, options)
}

  // strict mode
  this.strict = strict

  const state = this._modules.root.state

  // init root module.
  // this also recursively registers all sub-modules
  // and collects all module getters inside this._wrappedGetters
  installModule(this, state, [], this._modules.root)

  // 重点方法 ，重置VM
  resetStoreVM(this, state)

  // apply plugins
  plugins.forEach(plugin => plugin(this))

}
```

除了一堆初始化外，我们注意到了这样一行代码
 **resetStoreVM(this, state)** 他就是整个vuex的关键



```javascript
// src/store.js
function resetStoreVM (store, state, hot) {
  // 省略无关代码
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
}
```

去除了一些无关代码后我们发现，其本质就是将我们传入的state作为一个隐藏的vue组件的data,也就是说，我们的commit操作，本质上其实是修改这个组件的data值，结合上文的computed,修改被**defineReactive**代理的对象值后，会将其收集到的依赖的**watcher**中的**dirty**设置为true,等到下一次访问该watcher中的值后重新获取最新值。

这样就能解释了为什么vuex中的state的对象属性必须提前定义好，如果该**state**中途增加**一个属性**，因为该**属性**没有被**defineReactive**，所以其依赖系统没有检测到，自然不能更新。

由上所说，我们可以得知store._vm.$data.$$state === store.state, 我们可以在任何含有vuex框架的工程验证这一点。



![img](https:////upload-images.jianshu.io/upload_images/8032324-11306d44d5b577e4.png?imageMogr2/auto-orient/strip|imageView2/2/w/634/format/webp)

## 总结

vuex整体思想诞生于**flux**,可其的实现方式完完全全的使用了vue自身的响应式设计，依赖监听、依赖收集都属于vue对对象Property set get方法的代理劫持。最后一句话结束vuex工作原理，`vuex中的store本质就是没有`template`的隐藏着的vue组件；`



作者：Kaku_fe
链接：https://www.jianshu.com/p/d95a7b8afa06
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。