# [Vue源码学习] new Vue()

## 前言

在使用`Vue`编写应用程序时，都是通过用`Vue`构造函数创建一个新的`Vue`实例开始的：

```js
new Vue({
  // options
})
```

那么接下来，我们就来看看从入口开始，`Vue`内部做了哪些工作。

## Vue构造函数

我们首先来看看`Vue`构造函数的定义，其代码如下所示：

```js
/* core/instance/index.js */
function Vue(options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```

可以看到，当使用`new`运算符新建`Vue`实例时，其内部就是调用`_init`方法，其代码如下所示：

```js
/* core/instance/init.js */
Vue.prototype._init = function (options?: Object) {
  const vm: Component = this
  // a uid
  vm._uid = uid++

  // a flag to avoid this being observed
  vm._isVue = true

  // 1. 配置合并
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    initProxy(vm)
  } else {
    vm._renderProxy = vm
  }
  // expose real self
  vm._self = vm

  // 2. 初始化vm实例
  initLifecycle(vm)
  initEvents(vm)
  initRender(vm)
  callHook(vm, 'beforeCreate')
  initInjections(vm) // resolve injections before data/props
  initState(vm)
  initProvide(vm) // resolve provide after data/props
  callHook(vm, 'created')

  // 3. 挂载根节点
  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```

可以看到，在`_init`方法中，`Vue`主要做了三件事情：

1. 配置合并

    ```js
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    ```

    配置合并可以分为组件和非组件两种形式，`Vue`会根据这两种情况，分别调用`initInternalComponent`和`mergeOptions`方法，不管调用哪一种方法，其作用都是将传入的`options`和`Ctor.options`做合并操作，然后将合并后的结果保存在`vm.$options`中。

2. 初始化vm实例

    ```js
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
    ```

    在创建实例时，`Vue`会调用多个`init`方法，每个方法都会在实例上初始化一些数据。那么接下来，我们就简单看下各个方法都做了哪些初始化工作。

    * initLifecycle

      ```js
      /* core/instance/lifecycle.js */
      export function initLifecycle(vm: Component) {
        const options = vm.$options

        // 将子实例添加到父实例的$children中
        let parent = options.parent
        if (parent && !options.abstract) {
          while (parent.$options.abstract && parent.$parent) {
            parent = parent.$parent
          }
          parent.$children.push(vm)
        }

        // 保存对父实例的引用
        vm.$parent = parent
        vm.$root = parent ? parent.$root : vm

        vm.$children = []
        vm.$refs = {}

        vm._watcher = null
        vm._inactive = null
        vm._directInactive = false
        vm._isMounted = false
        vm._isDestroyed = false
        vm._isBeingDestroyed = false
      }
      ```

      在`initLifecycle`方法中，除了在当前实例上初始化一些实例属性外，还通过`$children`和`$parent`构建实例之间的父子关系，这里需要注意的是，对于抽象组件来说，它是不会出现在组件的父组件链中的，也就是说，在父实例的`$children`中，会越过抽象组件，直接保存抽象组件的子组件，该子组件的`$parent`也是直接指向抽象组件的父组件，就像抽象组件不存在一样。

    * initEvents

      ```js
      /* core/instance/events.js */
      export function initEvents(vm: Component) {
        vm._events = Object.create(null)
        vm._hasHookEvent = false

        // 绑定到vm实例上的事件
        const listeners = vm.$options._parentListeners
        if (listeners) {
          updateComponentListeners(vm, listeners)
        }
      }
      ```

      `initEvents`主要用来处理事件相关的内容，需要注意的是，这里的事件属于自定义事件，是绑定到当前实例上的，而不是`DOM`相关的事件。

    * initRender

      ```js
      /* core/instance/render.js */
      export function initRender(vm: Component) {
        vm._vnode = null // the root of the child tree
        vm._staticTrees = null // v-once cached trees
        const options = vm.$options

        // 1. 解析普通插槽
        const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
        const renderContext = parentVnode && parentVnode.context
        vm.$slots = resolveSlots(options._renderChildren, renderContext)
        vm.$scopedSlots = emptyObject

        // 2. 添加创建VNode的方法
        // bind the createElement fn to this instance
        // so that we get proper render context inside it.
        // args order: tag, data, children, normalizationType, alwaysNormalize
        // internal version is used by render functions compiled from templates
        vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
        // normalization is always applied for the public version, used in
        // user-written render functions.
        vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

        // 3. 在实例上定义响应式属性$attrs和$listeners
        // $attrs & $listeners are exposed for easier HOC creation.
        // they need to be reactive so that HOCs using them are always updated
        const parentData = parentVnode && parentVnode.data

        /* istanbul ignore else */
        if (process.env.NODE_ENV !== 'production') {
          defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
            !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
          }, true)
          defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
            !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
          }, true)
        } else {
          defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
          defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
        }
      }
      ```

      `initRender`首先会调用`resolveSlots`方法对普通插槽进行解析，然后在当前实例上添加`_c`和`$createElement`方法，这两个方法都与渲染相关，用来创建`VNode`，最后在当前实例上定义响应式属性`$attrs`和`$listeners`，从而在组件的父占位符节点做`updateChildComponent`时，可以很方便的通知组件实例做更新操作。

    * initInjections & initProvide

      ```js
      /* core/instance/inject.js */
      export function initInjections(vm: Component) {
        const result = resolveInject(vm.$options.inject, vm)
        if (result) {
          toggleObserving(false)
          Object.keys(result).forEach(key => {
            /* istanbul ignore else */
            if (process.env.NODE_ENV !== 'production') {
              // ...
            } else {
              defineReactive(vm, key, result[key])
            }
          })
          toggleObserving(true)
        }
      }

      export function resolveInject(inject: any, vm: Component): ?Object {
        // 配置中定义的inject选项，此时已经经过normalizeInject处理
        if (inject) {
          // inject is :any because flow is not smart enough to figure out cached
          const result = Object.create(null)
          const keys = hasSymbol
            ? Reflect.ownKeys(inject)
            : Object.keys(inject)

          for (let i = 0; i < keys.length; i++) {
            const key = keys[i]
            // #6574 in case the inject object is observed...
            if (key === '__ob__') continue
            const provideKey = inject[key].from
            let source = vm
            // 向上递归$parent，找到提供provideKey的祖先实例，然后将结果注入到当前实例中
            while (source) {
              if (source._provided && hasOwn(source._provided, provideKey)) {
                result[key] = source._provided[provideKey]
                break
              }
              source = source.$parent
            }
            if (!source) {
              // 如果在祖先中找不到provideKey，但是定义了default，那么就使用默认值，否则提示警告
              if ('default' in inject[key]) {
                const provideDefault = inject[key].default
                result[key] = typeof provideDefault === 'function'
                  ? provideDefault.call(vm)
                  : provideDefault
              } else if (process.env.NODE_ENV !== 'production') {
                warn(`Injection "${key}" not found`, vm)
              }
            }
          }
          return result
        }
      }
      ```

      ```js
      /* core/instance/inject.js */
      export function initProvide(vm: Component) {
        const provide = vm.$options.provide
        // 通过provide选项，向所有子孙后代注入依赖
        if (provide) {
          vm._provided = typeof provide === 'function'
            ? provide.call(vm)
            : provide
        }
      }
      ```

      可以看到，`provide`和`inject`选项需要搭配使用，在祖先中可以通过`provide`选项，为所有的后代注入依赖，不论组件嵌套的有多深，后代都可以通过`inject`选项，将祖先提供的数据注入到当前实例中。

    * initState

      ```js
      /* core/instance/state.js */
      export function initState(vm: Component) {
        vm._watchers = []
        const opts = vm.$options
        if (opts.props) initProps(vm, opts.props)
        if (opts.methods) initMethods(vm, opts.methods)
        if (opts.data) {
          initData(vm)
        } else {
          observe(vm._data = {}, true /* asRootData */)
        }
        if (opts.computed) initComputed(vm, opts.computed)
        if (opts.watch && opts.watch !== nativeWatch) {
          initWatch(vm, opts.watch)
        }
      }
      ```

      `initState`就是用来处理`props`、`methods`、`data`、`computed`、`watch`选项的，这些内容会在之后的章节中详细介绍。

3. 挂载根节点

    ```js
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
    ```

    在`_init`方法最后，如果配置中存在`el`选项，就会调用`$mount`方法进行挂载。

## 总结

在`new Vue()`创建实例的过程中，抛开最后`$mount`，其实就是创建一个`Vue`的实例，然后对其做一系列的初始化操作，比如在实例上添加各种属性、方法，处理各种配置选项，完成实例的初始化工作后，就可以调用`$mount`，对该实例进行挂载。
