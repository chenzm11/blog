# [Vue源码学习] 配置合并

## 前言

从上一章节中我们知道，在调用`_init`方法初始化`Vue`实例的过程中，首先会进行配置合并的操作：

```js
/* core/instance/init.js */
Vue.prototype._init = function (options?: Object) {
  if (options && options._isComponent) {
    // 组件的配置合并
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  } else {
    // 非组件的配置合并
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
}
```

可以看到，`Vue`会根据不同的条件，进行非组件的配置合并和组件的配置合并，那么接下来，我们就分别看看它们内部是如何实现的。

## 非组件的配置合并

当直接使用`new Ctor()`的方式创建实例时，就会调用`mergeOptions`方法，进行非组件的配置合并，代码如下所示：

```js
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

可以看到，`mergeOptions`方法接收三个参数，其中第一个参数是调用`resolveConstructorOptions`方法返回的，其内部逻辑之后会单独介绍，这里可以简单的看成返回`Ctor.options`，那么接下来，我们就来看看`mergeOptions`方法的实现：

```js
/* core/util/options.js */
export function mergeOptions(
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }

  // 1. 规范化props、inject、directives选项
  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)

  // 2. 将extends、mixins选项中的配置合并到parent中
  if (!child._base) {
    if (child.extends) {
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }

  // 3. 根据策略函数处理选项
  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField(key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```

可以看到，在`mergeOptions`方法中，首先调用`normalize`方法规范化`props`、`inject`、`directives`选项，将用户传入的数据处理成规范的格式；然后递归地调用`mergeOptions`方法，将`extends`、`mixins`选项中的配置合并到`parent`中，相当于对组件做了扩展，同时从这里可以看出，`extends`和`mixins`选项的处理逻辑是相同的，`mixins`相当于包含多次`extends`；在完成了前两步的初始化工作后，接下来就开始执行真正的配置合并的逻辑。

在合并的时候，首先遍历传入的配置对象，根据不同的选项从`strats`中取出对应的合并策略函数，然后将`parent`中的配置和`child`中的配置通过策略函数进行合并，最后就可以得到此选项合并后的结果了。接下来就来看看常见的选项是如何进行合并的。

### lifecycle

```js
/* shared/constants.js */
export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured',
  'serverPrefetch'
]

/* core/util/options.js */
LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})
```

可以看到，所有生命周期的策略函数都是`mergeHook`，其代码如下所示：

```js
/* core/util/options.js */
function mergeHook(
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  const res = childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
  return res
    ? dedupeHooks(res)
    : res
}

function dedupeHooks(hooks) {
  const res = []
  for (let i = 0; i < hooks.length; i++) {
    if (res.indexOf(hooks[i]) === -1) {
      res.push(hooks[i])
    }
  }
  return res
}
```

`mergeHook`会将`parent`和`child`中的生命周期钩子函数做合并操作，返回一个新的数组，其中`parent`中的钩子函数在前，`child`中的钩子函数在后，最后使用`dedupeHooks`方法过滤掉重复的钩子函数。

### components、directives、filters

```js
/* shared/constants.js */
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]

/* core/util/options.js */
ASSET_TYPES.forEach(function (type) {
  strats[type + 's'] = mergeAssets
})
```

可以看到，`components`、`directives`、`filters`选项的策略函数都是`mergeAssets`，其代码如下所示：

```js
/* core/util/options.js */
function mergeAssets(
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): Object {
  const res = Object.create(parentVal || null)
  if (childVal) {
    process.env.NODE_ENV !== 'production' && assertObjectType(key, childVal, vm)
    return extend(res, childVal)
  } else {
    return res
  }
}
```

`mergeAssets`首先会以`parentVal`为原型创建一个新的实例，然后把`childVal`合并到这个实例上，之所以使用原型继承的方式，是因为`components`、`directives`、`filters`这些资源相关的选项，在多个实例之间是可以复用的，这样就不用把这些数据单独的添加到各个实例上了。

### data、provide

```js
/* core/util/options.js */
strats.data = function (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  // ...
  return mergeDataOrFn(parentVal, childVal, vm)
}

strats.provide = mergeDataOrFn

export function mergeDataOrFn(
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  // ...
  return function mergedInstanceDataFn() {
    // instance merge
    const instanceData = typeof childVal === 'function'
      ? childVal.call(vm, vm)
      : childVal
    const defaultData = typeof parentVal === 'function'
      ? parentVal.call(vm, vm)
      : parentVal
    if (instanceData) {
      return mergeData(instanceData, defaultData)
    } else {
      return defaultData
    }
  }
}
```

可以看到，对于非组件的配置合并，调用`data`选项的策略函数会返回一个新的函数`mergedInstanceDataFn`，它会在`initState`的过程中执行。执行时，首先从`parent`和`child`中取到各自的数据，然后调用`mergeData`方法合并这两项数据：

```js
/* core/util/options.js */
function mergeData(to: Object, from: ?Object): Object {
  if (!from) return to
  let key, toVal, fromVal

  const keys = hasSymbol
    ? Reflect.ownKeys(from)
    : Object.keys(from)

  for (let i = 0; i < keys.length; i++) {
    key = keys[i]
    // in case the object is already observed...
    if (key === '__ob__') continue
    toVal = to[key]
    fromVal = from[key]
    if (!hasOwn(to, key)) {
      // 对于to中不存在的数据，从from中取出后添加到to中
      set(to, key, fromVal)
    } else if (
      toVal !== fromVal &&
      isPlainObject(toVal) &&
      isPlainObject(fromVal)
    ) {
      // 值是对象，需要递归处理
      mergeData(toVal, fromVal)
    }
  }
  return to
}
```

可以看到，`mergeData`就是一个深度合并的过程，主要是用来将`from`中的数据合并到`to`中，所以首先遍历`from`对象，如果`to`中不存在该数据，就将数据直接添加到`to`中；如果`to`中存在该数据，并且它们对应的值还是一个对象，就继续调用`mergeData`，对数据进行深度递归，否则，就不做任何操作，还是以`to`中的数据为准，最终可以得到一个以`to`为主的对象，这样就可以把`child`和`parent`中的数据合并起来了。

### props、methdos、inject、computed

```js
/* core/util/options.js */
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  if (childVal && process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = Object.create(null)
  extend(ret, parentVal)
  if (childVal) extend(ret, childVal)
  return ret
}
```

可以看到，这些合并策略很简单，就是浅合并`parentVal`和`childVal`，并且`childVal`中的值可以覆盖`parentVal`中的值。

### 默认策略

对于没有策略函数的选项来说，就会使用默认策略，代码如下所示：

```js
/* core/util/options.js */
const defaultStrat = function (parentVal: any, childVal: any): any {
  return childVal === undefined
    ? parentVal
    : childVal
}
```

默认策略就是如果在`child`中定义了选项，就直接取`child`中的值，否则，就取`parent`中的值。

在各种选项经过对应的策略函数处理后，将最终的合并结果赋值给`vm.$options`，这样就完成了非组件的配置合并。

那么接下来，就来看看对于组件来说，它是如何合并的。

## 组件的配置合并

一般来说，组件的配置合并首先会发生在创建组件`VNode`的过程中，在调用`createComponent`方法时，会调用`Vue.extend`方法，将组件配置对象转换成组件构造器，其代码如下所示：

```js
/* core/global-api/extend.js */
Vue.extend = function (extendOptions: Object): Function {
  extendOptions = extendOptions || {}
  const Super = this
  const SuperId = Super.cid
  const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
  // 首先尝试从缓存中获取组件构造器
  if (cachedCtors[SuperId]) {
    return cachedCtors[SuperId]
  }

  const name = extendOptions.name || Super.options.name
  if (process.env.NODE_ENV !== 'production' && name) {
    validateComponentName(name)
  }

  // 定义组件构造器
  const Sub = function VueComponent(options) {
    this._init(options)
  }
  Sub.prototype = Object.create(Super.prototype)
  Sub.prototype.constructor = Sub
  Sub.cid = cid++

  // 配置合并
  Sub.options = mergeOptions(
    Super.options,
    extendOptions
  )
  Sub['super'] = Super

  // For props and computed properties, we define the proxy getters on
  // the Vue instances at extension time, on the extended prototype. This
  // avoids Object.defineProperty calls for each instance created.
  if (Sub.options.props) {
    initProps(Sub)
  }
  if (Sub.options.computed) {
    initComputed(Sub)
  }

  // allow further extension/mixin/plugin usage
  Sub.extend = Super.extend
  Sub.mixin = Super.mixin
  Sub.use = Super.use

  // create asset registers, so extended classes
  // can have their private assets too.
  ASSET_TYPES.forEach(function (type) {
    Sub[type] = Super[type]
  })
  // enable recursive self-lookup
  if (name) {
    Sub.options.components[name] = Sub
  }

  // keep a reference to the super options at extension time.
  // later at instantiation we can check if Super's options have
  // been updated.
  Sub.superOptions = Super.options
  Sub.extendOptions = extendOptions
  Sub.sealedOptions = extend({}, Sub.options)

  // cache constructor
  cachedCtors[SuperId] = Sub
  return Sub
}
```

可以看到，`extend`方法首先会检查缓存中是否存在对应的子构造器，如果存在就直接返回，如果不存在，就先创建了一个子构造器`Sub`，然后使用原型继承的方式使`Sub`继承`Super`，接着调用`mergeOptions`方法，将`Super.options`和组件配置进行合并，然后将合并后的结果添加到`Sub.options`中，合并完成后，将`Super`上的静态方法添加到`Sub`上，比如`extend`、`mixin`、`use`、`component`、`directive`、`filter`等，此时，`Sub`构造器就拥有了类似`Super`构造器的能力，然后将`Sub`自己添加到`Sub.options.components`中，这样在组件中就可以递归调用自己。最后将构造出的子构造器`Sub`缓存到组件配置对象中，下一次就可以直接从缓存中取出子构造器，而不用重新构建了。

通过`Vue.extend`创建完子构造器后，此时还没有创建子组件的实例，当父组件执行`patch`，准备挂载组件的父占位符节点时，会调用`createComponentInstanceForVnode`方法创建子组件的实例，代码如下所示：

```js
/* core/vdom/create-component.js */
export function createComponentInstanceForVnode(
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode, // 父占位符节点
    parent // 父vm实例
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}
```

可以看到，在`createComponentInstanceForVnode`方法中，首先会构建组件的配置选项，`_isComponent`表示当前是组件的配置合并，其余的`_parentVnode`和`parent`分别表示组件的父占位符节点和父`vm`实例，然后通过`vnode.componentOptions.Ctor`，也就是上面的`Sub`构造器，创建子组件的实例，最终还是调用`_init`方法，只是这次传入的选项是`Vue`内部构造的，接下来，我们来看看对于组件的配置合并，`_init`内部是如何工作的：

```js
/* core/instance/init.js */
if (options && options._isComponent) {
  // optimize internal component instantiation
  // since dynamic options merging is pretty slow, and none of the
  // internal component options needs special treatment.
  initInternalComponent(vm, options)
}
```

可以看到，对于组件来说，会调用`initInternalComponent`方法进行配置合并，代码如下所示：

```js
/* core/instance/init.js */
export function initInternalComponent(vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```

在`initInternalComponent`方法中，首先以`Sub.options`为原型创建一个实例，然后将各种与当前实例相关的`propsData`、`listeners`、`children`等属性添加到该实例上，可以看到，在创建子组件实例时，由于没有做`mergeOptions`操作，所以配置合并的速度是很快的。

## resolveConstructorOptions

在前面的两小节中，我们已经知道`Vue`是如何合并配置的了，其实在它们的内部，都会执行`resolveConstructorOptions`方法，用来检查构造器上的`options`是否需要更新，代码如下所示：

```js
/* core/instance/init.js */
export function resolveConstructorOptions(Ctor: Class<Component>) {
  let options = Ctor.options
  if (Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    // 缓存的配置和实际的配置不同时，需要重新做mergeOptions操作
    if (superOptions !== cachedSuperOptions) {
      // super option changed,
      // need to resolve new options.
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options (#4976)
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      }
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}
```

在`resolveConstructorOptions`方法中，如果检测到子构造器中的`superOptions`与父构造器的`options`不相同，则说明在创建完子构造器后，父构造器的配置选项进行过更新，所以需要调用`mergeOptions`方法，重新创建子构造器的`options`，所以在创建实例之前，通过调用`resolveConstructorOptions`方法，就可以保证在创建实例时，构造器上的`options`选项是最新的了。

## 总结

在`Vue`中，每个实例的`$options`不仅仅来自于我们编写的配置，它还会用不同的策略函数，与组件构造器上的配置进行合并，同时，子组件只会在第一次构造组件构造器时执行`mergeOptions`操作，之后就可以高效的创建子组件的实例了。
