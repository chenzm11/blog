# [Vue源码学习] 响应式原理（上）

## 前言

在前几章节中，我们已经可以根据组件模板，渲染成真实的`DOM`，而响应式系统则可以控制渲染的逻辑和实际内容，同时收集在渲染过程中访问到的数据，在数据发生变化时，通知所有的观察者，从而使它关联的组件进行重新渲染，那么接下来，就来看看`Vue`中的响应式系统是如何工作的。

## initData

在初始化`Vue`实例的过程中，会调用`initState`方法，代码如下所示：

```js
/* core/instance/init.js */
Vue.prototype._init = function (options?: Object) {
  // ...
  initState(vm)
  // ...
}

/* core/instance/state.js */
export function initState(vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // ...
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  // ...
}
```

可以看到，如果配置选项中存在`data`选项，就会调用`initData`方法进行处理，否则会创建一个空对象，然后调用`observe`方法，其实`initData`方法内部同样是调用`observe`方法，只不过多了一些校验和代理的逻辑，`initData`方法的代码如下所示：

```js
/* core/instance/state.js */
function initData(vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}

/* core/util/lang.js */
export function isReserved(str: string): boolean {
  const c = (str + '').charCodeAt(0)
  return c === 0x24 || c === 0x5F
}
```

可以看到，在`initData`方法中，首先通过`$options.data`获取到真实的数据，并赋值给`vm._data`；然后获取该实例上已经定义的`props`和`methods`，接着遍历`data`中的数据，对已经存在于`props`或`methods`中的数据，在开发模式下会提示警告，对于不存在的数据，并且属性名不是以`$`或`_`开头，就调用`proxy`方法，将当前数据代理到当前实例上，`proxy`方法的代码如下所示：

```js
/* core/instance/state.js */
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}

export function proxy(target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter() {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter(val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

经过`proxy`方法处理后，就可以使用`this[key]`代替原来的`this._data[key]`了。在`initData`方法的最后，还是调用`observe`方法，进行数据的响应式处理，那么接下来就来看看`observe`方法的具体实现。

## observe

`observe`方法的代码如下所示：

```js
/* core/observer/index.js */
export function observe(value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

可以看到，`observe`方法的逻辑是很清晰的。首先只有当`value`是对象并且不是`VNode`的实例时，才允许创建响应式；对于已经创建过响应式的对象，可以直接通过`value.__ob__`进行访问，对于普通的对象，在满足一定的条件下，会调用`Observer`构造函数，创建响应式对象；接下来的`ob.vmCount++`是用来防止`Vue`实例重复使用同一个`data`选项，确保每个实例可以维护一份自己的数据；最后返回`ob`对象。接下来，就来看看`Vue`是如何通过`Observer`构造函数将普通对象转换成响应式对象的。

## Observer

`Observer`构造函数的代码如下所示：

```js
/* core/observer/index.js */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor(value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk(obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray(items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

可以看到，每一个`Observer`对象都包含一个`Dep`的实例，它是用来连接目标对象和观察者的管理器，在下一小节中会详细介绍`Dep`的作用，然后使用`def`方法，将`Observer`实例保存在`value`的`__ob__`属性中，代表着`value`已经经过响应式处理，不用再次重新`new Observer`；接下来就开始处理传入的数据了，需要处理的数据分为普通对象和数组，由于结构的不同，使用的处理方式也不一样，下面分别来看看它们是如何进行处理的。

### 普通对象

对于普通对象而言，`Vue`会通过`walk`方法进行处理，代码如下所示：

```js
/* core/observer/index.js */
export class Observer {
  walk(obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }
}
```

可以看到，`walk`方法就是遍历传入的数据，然后调用`defineReactive`方法，从而将普通数据变为响应式数据，其代码如下所示：

```js
/* core/observer/index.js */
export function defineReactive(
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      // ...
    },
    set: function reactiveSetter(newVal) {
      // ...
    }
  })
}
```

可以看到，在`defineReactive`方法中，`Vue`也同样创建了一个`Dep`的实例，与每个目标属性相连；接下来是处理不可配置属性和访问器属性的逻辑；然后在满足`shallow`参数为假的情况下，会继续调用`observe`方法，递归的处理深层嵌套对象，从而使整个对象的每一层级，都变成响应式的；最后就是通过`Object.defineProperty`方法，将属性定义为访问器属性，从而实现数据的响应式化，其实就是在访问或设置属性的过程中，添加一个中间层，从而执行一些额外的逻辑。需要注意的是，定义响应式的代码是在初始化`Vue`实例时执行的，此时并没有访问或设置属性，也就是没有执行上面的`get`和`set`访问器，在下一小节中，会详细介绍它们是如何工作的。那么接下来，就看看`Vue`是如何处理数组的。

### 数组

对于数组来说，`Vue`执行的逻辑如下所示：

```js
/* core/util/env.js */
export const hasProto = '__proto__' in {}

/* core/observer/index.js */
export class Observer {
  constructor(value: any) {
    // ...
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    }
    // ...
  }

  observeArray(items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}

function protoAugment(target, src: Object) {
  /* eslint-disable no-proto */
  target.__proto__ = src
  /* eslint-enable no-proto */
}

function copyAugment(target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```

可以看到，`Vue`处理数组的逻辑其实有两步，第一步是在数组上添加变异方法`arrayMethods`，第二步是调用`observe`方法，递归处理数组中的每一项数据，`observe`方法的逻辑在上面已经介绍过，那么接下来，就来看看变异方法`arrayMethods`是如何创建的。

在`core/observer/array.js`中，包含了`arrayMethods`的全部逻辑，代码如下所示：

```js
/* core/observer/array.js */
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)

const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator(...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify()
    return result
  })
})
```

可以看到，`arrayMethods`首先是继承自`Array.prototype`的，所以它拥有数组原型上的所有原生方法，然后给`arrayMethods`对象上重新定义`push`、`pop`、`shift`、`unshift`、`splice`、`sort`、`reverse`这7个方法，当该数组调用这些变异方法时，首先会执行变异方法的默认操作，然后会从该数组上获取`__ob__`属性，也就是之前定义的`Observer`对象，对于`push`、`unshift`、`splice`方法而言，由于它们可以给数组添加新的数据，所以首先需要使用`ob.observeArray`方法，将它们处理成响应式数据，最后所有的变异方法都会调用`ob.dep.notify`方法，这段逻辑是变异方法中最关键的逻辑，通过调用`notify`方法，可以通知所有与之相关的观察者，从而完成组件的重新渲染。

## 总结

`Vue`实例在初始化的过程中，会调用`initData`方法，处理配置中的`data`选项，通过`Observer`构造函数和`defineReactive`方法，就可以将整个对象通过`Object.defineProperty`方法，处理成响应式数据，并且对于数组来说，`Vue`对7个方法进行了扩展，使之能够触发响应式。
