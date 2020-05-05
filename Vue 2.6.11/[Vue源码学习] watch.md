# [Vue源码学习] watch

## 前言

`Vue`通过`watch`选项，提供了一种更通用的方式来观察和响应`Vue`实例上的数据变动，那么接下来，就来看看在`Vue`中，是如何使用`watch`选项的。

## watch

在初始化`Vue`实例的过程中，如果检测到配置中存在`watch`选项，就会调用`initWatch`方法，处理侦听属性，代码如下所示：

```js
/* core/instance/state.js */
export function initState(vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // ...
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}

function initWatch(vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```

可以看到，在`initWatch`方法中，就是遍历`watch`选项，继续调用`createWatcher`方法，做进一步的处理，代码如下所示：

```js
/* core/instance/state.js */
function createWatcher(
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

可以看到，`createWatcher`方法就是用来规范化传入的参数，在得到`handler`处理函数后，就去调用`$watch`方法，所以在`watch`选项中定义的侦听属性，与手动调用`$watch`方法的作用是相同的。

## $watch

`$watch`方法是在引入`Vue`时，添加到`Vue`的原型上的，代码如下所示：

```js
/* core/instance/state.js */
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {
  const vm: Component = this
  if (isPlainObject(cb)) {
    return createWatcher(vm, expOrFn, cb, options)
  }
  options = options || {}
  options.user = true
  const watcher = new Watcher(vm, expOrFn, cb, options)
  if (options.immediate) {
    try {
      cb.call(vm, watcher.value)
    } catch (error) {
      handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
    }
  }
  return function unwatchFn() {
    watcher.teardown()
  }
}
```

可以看到，在`$watch`方法中，首先在`options`上添加`user`属性，表明这是一个`user watcher`，然后创建`Watcher`实例，与之前章节中创建`Watcher`实例类似，但是对于自定义`Watcher`来说，它会传入一个在数据改变时调用的回调函数，还可以传入自定义配置选项，比如`deep`、`sync`等，并且以前的渲染`Watcher`和计算`Watcher`，它们的`expOrFn`参数都是一个函数，而自定义`Watcher`除了传入函数以外，还可以是一个字符串，它会经过`parsePath`方法处理后，再赋值给`getter`属性，代码如下所示：

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
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    // ...
  }
}

/* core/util/lang.js */
const bailRE = new RegExp(`[^${unicodeRegExp.source}.$_\\d]`)
export function parsePath(path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```

可以看到，`watcher.getter`指向了`parsePath`返回的匿名函数。回到`Watcher`的构造函数中，如果此时`lazy`为`false`，那么就会继续调用`get`方法，进行依赖收集，这时，就会调用`getter`方法，也就是`parsePath`中的匿名函数，此时，匿名函数的参数`obj`就是当前`Vue`实例，`segments`就是通过`.`分割字符串后得到的片段数组，然后遍历此数组，从而获取最终的值，从这里也可以看出，对于当前还不存在的数据，进行依赖收集时是不会报错的。

执行完`watcher.get`方法后，将待侦听的属性对应的值保存在`watcher.value`中，并且此表达式与自定义`Watcher`之间，已经成功构建了联系。回到`$watch`方法中，最后会返回一个`unwatchFn`方法，用来取消观察，调用此方法时，会调用`watcher.teardown`方法，代码如下所示：

```js
/* core/observer/watcher.js */
export default class Watcher {
  teardown() {
    if (this.active) {
      // remove self from vm's watcher list
      // this is a somewhat expensive operation so we skip it
      // if the vm is being destroyed.
      if (!this.vm._isBeingDestroyed) {
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}
```

可以看到，在`teardown`方法中，首先对此`Watcher`所依赖的数据，通过调用`dep.removeSub`方法进行取消依赖，然后将此`Watcher`的`active`选项置为`false`，从而取消观察，这样一来，当侦听的属性发生变化时，就不会触发该`Watcher`的更新了。

接下来，就来看看数据发生变化时，`Vue`是如何处理的。

## update

当观察的数据发生变化时，就会通知此`Watcher`调用它的`update`方法进行更新，对于非`sync`的`Watcher`来说，同样也会将`Watcher`实例推入`queue`中，然后在下一帧中执行`watcher.run`方法，代码如下所示：

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

  run() {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }
}
```

可以看到，在`watcher.run`方法中，首先还是会调用`watcher.get`方法，重新收集依赖，得到更新之后的值，如果和上一次的`value`不相同时，就会调用`watcher.cb`方法，也就是调用`$watch`时传入的`handle`处理函数，同时将新旧`value`作为参数传入，所以我们定义的回调函数才可以执行。这就是自定义`Watcher`更新时的逻辑。

除了默认操作外，自定义`Watcher`还可以接收一些额外的选项，那么接下来，就来看看这些选项是如何运作的。

## immediate

在调用`$watch`创建自定义`Watcher`后，如果`immediate`选项为`true`，就会立即调用一次`cb`回调函数，代码如下所示：

```js
/* core/instance/state.js */
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {
  // ...
  const watcher = new Watcher(vm, expOrFn, cb, options)
  if (options.immediate) {
    try {
      cb.call(vm, watcher.value)
    } catch (error) {
      handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
    }
  }
  // ...
}
```

## deep

在调用`watcher.get`进行依赖收集时，如果`deep`选项为`true`，就会调用`traverse`方法，进行深度依赖，代码如下所示：

```js
/* core/observer/watcher.js */
export default class Watcher {
  get() {
    // ...
    if (this.deep) {
      traverse(value)
    }
    // ...
  }
}

/* core/observer/traverse.js */
const seenObjects = new Set()

export function traverse(val: any) {
  _traverse(val, seenObjects)
  seenObjects.clear()
}

function _traverse(val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  if (val.__ob__) {
    const depId = val.__ob__.dep.id
    if (seen.has(depId)) {
      return
    }
    seen.add(depId)
  }
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```

在调用`traverse`方法时，此时的`Dep.target`还是指向当前的`Watcher`实例，然后调用`_traverse`方法，深度递归的对碰触到的每个数据都进行依赖收集，并且还使用`seenObjects`避免循环依赖，最终，此`Watcher`会依赖所有碰触到的数据，任意层次的数据发生变化时，都会通知此`Watcher`进行更新。

## sync

在`Watcher`需要进行更新时，如果`sync`选项为`true`，不会将此`Watcher`添加到`queue`中，而是直接执行`watcher.run`方法，进行更新操作，代码如下所示：

```js
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

## 总结

在`Vue`中，可以使用`watch`选项或`$watch`方法，对属性进行侦听，当这些数据发生变化时，会执行我们传入的回调函数，并且自定义`Watcher`还可以支持多种配置选项，从而实现不同的功能。
