# [Vue源码学习] computed

## 前言

在`Vue`中可以使用计算属性，缓存中间的计算结果，只有在相关响应式依赖发生变化时，它们才会重新求值，从而避免重复的计算，提高性能，那么接下来，就来看看计算属性在`Vue`中是如何实现的。

## computed

在初始化`Vue`实例的过程中，会调用`initState`方法处理数据，在该方法中，如果检测到有`computed`选项，就会调用`initComputed`方法，处理计算属性，代码如下所示：

```js
/* core/instance/state.js */
export function initState(vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // ...
  if (opts.computed) initComputed(vm, opts.computed)
  // ...
}

const computedWatcherOptions = { lazy: true }

function initComputed(vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```

可以看到，在`initComputed`方法中，首先定义一个空对象`_computedWatchers`，用来存放计算属性相关的`Watcher`实例；然后遍历`computed`选项，对计算属性进行处理。

因为计算属性支持单个函数，也支持带`get`、`set`属性的对象，所以首先取得计算属性的取值函数`getter`，然后在非服务端渲染的情况下，创建计算属性对应的`Watcher`实例，代码如下所示：

```js
/* core/observer/watcher.js */
export default class Watcher {
  constructor(
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    // ...
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    // ...
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      // ...
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }
}
```

可以看到，在创建`Watcher`实例的过程中，这里的`getter`就是上面计算属性的`getter`，而`options`就是`computedWatcherOptions`，它的`lazy`选项为`true`，所以`Watcher`实例的`lazy`和`dirty`属性为`true`，由于`lazy`为`true`，所以对于计算`Watcher`来说，在创建时不会调用`get`方法进行求值。

回到上面的`initComputed`方法中，在创建好`Watcher`后，将其添加到`_computedWatchers`中。接着判断计算属性是否已经存在于当前`Vue`实例上，如果不存在，就调用`defineComputed`方法，将其添加到当前`Vue`实例上，代码如下所示：

```js
/* core/instance/state.js */
export function defineComputed(
  target: any,
  key: string,
  userDef: Object | Function
) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
  }
  if (process.env.NODE_ENV !== 'production' &&
    sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        `Computed property "${key}" was assigned to but it has no setter.`,
        this
      )
    }
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

可以看到，在`defineComputed`方法中，给当前`Vue`实例上添加新的访问器属性，其属性名是计算属性的属性名，而在非服务端渲染的情况下，它的`get`访问器是通过调用`createComputedGetter`方法创建的，其代码如下所示：

```js
/* core/instance/state.js */
function createComputedGetter(key) {
  return function computedGetter() {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
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

可以看到，`createComputedGetter`方法就是简单的返回了一个新函数`computedGetter`，而这个新函数就是对应计算属性的`get`访问器，所以当我们访问该计算属性时，就会执行`computedGetter`方法，那么接下来，就来看看在访问计算属性时，`Vue`又做了哪些工作。

## computedGetter

当访问计算属性时，会触发计算属性的`get`访问器，也就是`computedGetter`方法。在该方法中，首先会从`_computedWatchers`中取出对应的`Watcher`实例，然后通过`watcher.dirty`属性判断该计算属性是否需要重新计算，首次访问时，`dirty`属性为`true`，所以会调用`evaluate`方法，代码如下所示：

```js
/* core/observer/watcher.js */
export default class Watcher {
  get() {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

  evaluate() {
    this.value = this.get()
    this.dirty = false
  }
}
```

可以看到，在`evaluate`方法中，首先会调用`watcher.get`方法，在调用`pushTarget`方法后，会将`Dep.target`指向当前的计算`Watcher`，然后执行`getter`函数，也就是我们在配置选项中编写的函数，如果此计算属性依赖于`data`选项中的数据，那么就会触发该数据的`get`访问器，所以当前的计算`Watcher`会将该数据添加到自己的依赖中，同时此数据对应的`dep`也会将计算`Watcher`添加到它的观察者列表中，接下来的`popTarget`和`cleanupDeps`方法的逻辑就和之前相同，最终，将`watcher.get`方法返回的值赋值给`watcher.value`，然后将`dirty`置为`false`，表示当前计算属性已经是最新值，不用重新计算。

在执行完`evaluate`方法后，此时计算属性与它所依赖的数据之间就产生了联系，修改所依赖的数据时，就会通知计算属性进行更新。但是在`computedGetter`方法中，除了调用`evaluate`方法外，还有一段逻辑，如果此时`Dep.target`存在，就会调用`watcher.depend`方法，再进行一次依赖收集，可以想到这样的一个场景，在组件的渲染过程中，`Dep.target`首先会指向渲染`Watcher`，当访问计算属性时，会将渲染`Watcher`推入`targetStack`栈中，调用完`getter`方法后，又会将`Dep.target`指回渲染`Watcher`，由于此时`Dep.target`存在，所以就会执行`watcher.depend`方法，代码如下所示：

```js
/* core/observer/watcher.js */
export default class Watcher {
  depend() {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }
}

/* core/observer/dep.js */
export default class Dep {
  depend() {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
}
```

可以看到，在`depend`方法中，这里的`this`指向的还是计算`Watcher`，`this.deps`中保存的是计算属性依赖的数据，当执行`dep.depend`方法时，这里的`Dep.target`指向的却是渲染`Watcher`，所以又会在渲染`Watcher`和数据之间构建联系。

最终，在访问计算属性时，计算`Watcher`和渲染`Watcher`都会收集对此数据的依赖，所以当此数据发生变化时，会同时通知计算`Watcher`和渲染`Watcher`进行更新操作。那么接下来，就来看看计算`Watcher`是如何进行更新的。

## update

当依赖的数据发生变化时，就会遍历观察者集合，调用它们的`update`方法，代码如下所示：

```js
/* core/observer/watcher.js */
export default class Watcher {
  update() {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
}
```

可以看到，对于计算`Watcher`来说，由于它的`lazy`选项为`true`，所以它只会将`dirty`属性置为`true`，表示该计算属性需要重新进行计算，但不会将计算`Watcher`推入`watcher queue`中。这么做的好处是，如果更新后的渲染`Watcher`不依赖于该计算属性，就不用执行计算属性的重新求值了，只有当下次又访问到该计算属性时，由于在之前已经将`dirty`属性为`true`，所以才会执行计算属性的重新计算。

## 总结

在`Vue`中，可以通过计算属性对计算的结果进行缓存，避免重复计算的开销。在计算`Watcher`收集依赖的过程中，会将依赖的数据同步代理到上级`Watcher`中，所以在数据发生变化时，才会执行组件的重新渲染。
