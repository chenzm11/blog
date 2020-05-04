# [Vue源码学习] $mount挂载

## 前言

在调用`_init`方法初始化`Vue`实例的最后，如果选项中存在`el`属性，`Vue`就会直接调用`$mount`方法，进行挂载逻辑，代码如下所示：

```js
/* core/instance/init.js */
Vue.prototype._init = function (options?: Object) {
  // ...
  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```

那么接下来，就来看看对于运行时版本来说，`$mount`是如何进行挂载的。

## $mount

`$mount`方法其代码如下所示：

```js
/* platforms/web/runtime/index.js */
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

可以看到，在`$mount`方法中，首先将`el`选项转换成真实的`DOM`元素，然后调用`mountComponent`方法，其代码如下所示：

```js
/* core/instance/lifecycle.js */
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
    if (process.env.NODE_ENV !== 'production') {
      /* istanbul ignore if */
      if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
        vm.$options.el || el) {
        warn(
          'You are using the runtime-only build of Vue where the template ' +
          'compiler is not available. Either pre-compile the templates into ' +
          'render functions, or use the compiler-included build.',
          vm
        )
      } else {
        warn(
          'Failed to mount component: template or render function not defined.',
          vm
        )
      }
    }
  }
  callHook(vm, 'beforeMount')

  let updateComponent

  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    before() {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```

可以看到，`mountComponent`方法首先会检测配置中是否存在`render`函数，如果不存在会提示相应的警告，然后调用`beforeMount`钩子函数，接着定义`updateComponent`函数，这个函数是整个`Vue`渲染和更新的核心方法，之后会详细介绍，然后创建渲染`Watcher`，需要注意的是，每个`Vue`实例，有且仅有一个渲染`Watcher`，通过渲染`Watcher`，`Vue`可以完成依赖收集和派发更新的整个过程，`Watcher`构造函数的代码如下所示：

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
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
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
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
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
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
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
}
```

可以看到，在创建渲染`Watcher`的过程中，都会将该`Watcher`实例保存在组件的`_watcher`属性中，然后在该`Watcher`实例上添加一些实例属性，这里比较关键的就是将刚刚定义的`updateComponent`方法，赋值给渲染`Watcher`的`getter`属性，在创建渲染`Watcher`的最后，由于渲染`Watcher`的`lazy`为`false`，所以会直接调用`get`方法，在`get`方法中，会先后调用两个关键的方法`pushTarget`和`popTarget`，我们先来看看这两个方法的实现，其代码如下所示：

```js
/* core/observer/dep.js */
Dep.target = null
const targetStack = []

export function pushTarget(target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget() {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

可以看到，`targetStack`就是一个`watcher`实例的栈，用来存储各种`watcher`实例，调用`pushTarget`时会将当前`watcher`实例入栈，并将其赋值给`Dep.target`，这样在其他地方就可以通过`Dep.target`访问到当前最顶层的`watcher`实例，调用`popTarget`时会将最顶层的`watcher`实例出栈，并将上一个`watcher`实例赋值给`Dep.target`，通过这两个操作，就可以保证`watcher`的执行顺序。

回到上面的`get`方法，首先会调用`pushTarget`方法，将当前渲染`watcher`入栈并赋值给`Dep.target`，那么此时就可以在依赖收集的过程中，直接通过`Dep.target`访问到当前的渲染`watcher`。在设置好`Dep.target`后，就会调用`watcher.getter`方法，也就是上面定义的`updateComponent`方法，在该方法中，`Vue`首先会调用`_render`函数生成`VNode`，然后使用`_update`方法根据`VNode`渲染成真正的`DOM`并挂载到页面上，这部分内容会在之后的章节中详细介绍。在调用完`updateComponent`方法后，就会调用`popTarget`方法，恢复`Dep.target`的指向，最后调用`cleanupDeps`方法，进行清理工作，此时，渲染`Watcher`也已经收集到了所有它需要依赖的数据，这样整个初始渲染的工作就完成了，接着回到上面的`mountComponent`方法，如下：

```js
/* core/instance/lifecycle.js */
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  // ...

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```

可以看到，这里的`vm.$vnode`表示当前`Vue`实例的父占位符`VNode`，而对于根组件来说，它就是最顶层的节点，所以此时`vm.$vnode`为空，就会将`_isMounted`设置为`true`，表明当前`Vue`实例已经挂载，然后调用`mounted`钩子函数。此时，整个`$mount`方法所作的初始渲染工作就完成了，并且由于在调用渲染函数的过程中，渲染`Watcher`已经收集到了所有它需要依赖的数据，所以当这些数据发生变化时，就会触发`Watcher`的`update`操作，最终又会调用`getter`也就是`updateComponent`方法，重新做依赖收集和更新的操作，这部分内容在之后的章节中再详细介绍。

## 总结

在`Vue`中，实例的初始化和挂载是两个独立的模块，可以通过调用`$mount`方法进行挂载，在此过程中会为每个`Vue`实例创建唯一的一个渲染`Watcher`，同时将`updateComponent`方法赋值给渲染`Watcher`的`getter`属性，在创建渲染`Watcher`的过程中，会直接调用一次`getter`方法，完成依赖收集和首次挂载的逻辑，当它所依赖的数据发生变化时，`Watcher`又会重新执行`getter`方法，重新收集依赖，并执行对比更新操作。
