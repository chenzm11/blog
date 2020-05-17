# [Vue源码学习] new Vue()

## 前言

通常我们会使用`new Vue(options)`的方式来初始化`Vue`实例，那么接下来就来看看在`new Vue()`的过程中，`Vue`内部都做了哪些工作。首先来看看`Vue`构造函数是如何定义的。

## Vue构造函数

`Vue`构造函数的代码如下所示：

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

可以看到，当使用`new`运算符新建`Vue`实例时，其内部就是调用`_init`方法，同时将用户定义的选项`options`传入。接下来就来看看`_init`方法是如何定义的，其代码如下所示：

```js
/* core/instance/init.js */
Vue.prototype._init = function (options?: Object) {
  const vm: Component = this
  // a uid
  vm._uid = uid++

  // ...

  // a flag to avoid this being observed
  vm._isVue = true
  // merge options
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
  initLifecycle(vm)
  initEvents(vm)
  initRender(vm)
  callHook(vm, 'beforeCreate')
  initInjections(vm) // resolve injections before data/props
  initState(vm)
  initProvide(vm) // resolve provide after data/props
  callHook(vm, 'created')

  // ...

  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```

可以看到，在`_init`方法中，除了在当前实例上添加一些属性外，`Vue`主要做了三件事情：

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

    如上所示，`Vue`会根据不同的调用方式，分别执行`initInternalComponent`和`mergeOptions`方法，不管调用哪一种方法，都是将传入的选项`options`和`Ctor.options`做合并操作，最终将合并后的结果存储在`vm.$options`上，具体的合并细节会在之后的章节中详细介绍。

2. 执行多个`init`方法，进一步初始化`Vue`实例

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

    如上所示，`Vue`会调用多个`init`方法，并且在不同的阶段，还执行了`beforeCreate`、`created`生命周期钩子。那么接下来，就来简单的看下各个方法都做了哪些初始化工作。

    * initLifecycle

      ```js
      /* core/instance/lifecycle.js */
      export function initLifecycle(vm: Component) {
        const options = vm.$options

        // locate first non-abstract parent
        let parent = options.parent
        if (parent && !options.abstract) {
          while (parent.$options.abstract && parent.$parent) {
            parent = parent.$parent
          }
          parent.$children.push(vm)
        }

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

      如上所示，`initLifecycle`方法主要就是在当前实例上添加一些属性，并且通过`$parent`和`$children`建立组件之间的父子关系。这里的`$options.parent`是在合并配置时赋值的，表示当前实例的父`Vue`实例，对于非抽象组件来说，它会通过`while`循环，找到第一个非抽象组件的父实例，然后将当前实例添加到父实例的`$children`中，最后将父实例赋值给当前实例的`$parent`，这样就在`Vue`实例之间建立了父子关系。从这段逻辑可以看出，对于抽象组件来说，在父组件的`$children`中，无法访问到该抽象组件；对于普通组件来说，它会跨越抽象组件，将其`$parent`属性指向第一个非抽象父组件，同时也可以在该父组件的`$children`中访问到该普通组件的实例。

    * initEvents

      ```js
      /* core/instance/events.js */
      export function initEvents(vm: Component) {
        vm._events = Object.create(null)
        vm._hasHookEvent = false
        // init parent attached events
        const listeners = vm.$options._parentListeners
        if (listeners) {
          updateComponentListeners(vm, listeners)
        }
      }
      ```

      如上所示，`initEvents`方法主要用来处理事件相关的内容，需要注意的是，这里的事件属于自定义事件，是绑定到当前实例上的，而不是`DOM`相关的事件，在之后的章节中会详细介绍`Vue`是如何处理自定义事件和`DOM`事件的。

    * initRender

      ```js
      /* core/instance/render.js */
      export function initRender(vm: Component) {
        vm._vnode = null // the root of the child tree
        vm._staticTrees = null // v-once cached trees
        const options = vm.$options
        const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
        const renderContext = parentVnode && parentVnode.context
        vm.$slots = resolveSlots(options._renderChildren, renderContext)
        vm.$scopedSlots = emptyObject
        // bind the createElement fn to this instance
        // so that we get proper render context inside it.
        // args order: tag, data, children, normalizationType, alwaysNormalize
        // internal version is used by render functions compiled from templates
        vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
        // normalization is always applied for the public version, used in
        // user-written render functions.
        vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

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

      如上所示，`initRender`方法同样也是在当前实例上添加一些属性，该方法首先处理普通插槽`slot`相关的内容，然后在当前实例上添加了`vm._c`和`vm.$createElement`方法，这两个方法都与渲染相关，用来生成`VNode`，最后在当前实例上定义了响应式属性`$attrs`和`$listeners`，从而在父占位符节点更新的过程中，可以通知当前实例，`$attrs`和`$listeners`的变化。各功能的具体细节会在之后的章节中详细介绍。

    剩余的三个方法`initInjections`、`initProvide`、`initState`，会在之后的章节中再详细介绍。

    经过以上的两个步骤，`_init`方法的主要逻辑(初始化`Vue`实例)已经完成了。

3. 检测选项中是否存在`el`属性，如果存在就直接调用`$mount`方法进行挂载

    ```js
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
    ```

    如上所示，`_init`方法最后会判断用户是否传入了`el`选项，如果存在，就直接调用`$mount`方法进行挂载，其实这与手动调用`$mount`方法没有任何区别，只是`Vue`在初始化时帮我们做了这部分工作，`Vue`进行挂载的具体逻辑，会在之后的章节中详细介绍。

## 总结

本章简单的介绍了在`new Vue()`的过程中，`Vue`内部做了哪些初始化工作，其中主要是合并配置和执行`init`方法，在完成初始化工作后，如果选项中存在`el`属性，就会调用`$mount`方法进行挂载。
