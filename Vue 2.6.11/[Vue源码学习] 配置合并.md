# [Vue源码学习] 配置合并

## 前言

从上一章节中，我们知道，在调用`_init`方法初始化`Vue`实例的过程中，首先会进行配置合并的操作：

```js
/* core/instance/init.js */
Vue.prototype._init = function (options?: Object) {
  // ...

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

  // ...
}
```

如上所示，`Vue`会根据不同的条件，进行非组件的配置合并和组件的配置合并，那么接下来，就看看它们内部是如何合并配置的。

## 非组件的配置合并

当直接使用`new Ctor()`的方式创建实例时，会调用`mergeOptions`方法，进行非组件的配置合并，代码如下所示：

```js
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

可以看到，`mergeOptions`方法接收三个参数，其中第一个参数是调用`resolveConstructorOptions`方法返回的，其内部逻辑下面会介绍，这里可以简单看成返回`Ctor.options`，所以`mergeOptions`方法接收`Ctor`构造器上预定义的配置和传入的选项`options`，那么接下来，来看看`mergeOptions`方法的逻辑，代码如下所示：

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

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)

  // Apply extends and mixins on the child options,
  // but only if it is a raw options object that isn't
  // the result of another mergeOptions call.
  // Only merged options has the _base property.
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

可以看到，在`mergeOptions`方法中，首先规范化`props`、`inject`、`directives`选项，用于支持传入多种数据格式，方便`Vue`之后的处理流程；接着检测配置中是否包含`extends`、`mixins`选项，如果存在的话，就继续调用`mergeOptions`方法，将其中包含的选项合并到`parent`中，同时从这里可以看到，`extends`和`mixins`选项的处理逻辑是相同的，`mixins`相当于包含多次`extends`；在完成了前两步的初始化工作后，接下来就开始执行真正的配置合并的逻辑，因为`Vue`支持各种不同的选项，而每种选项的使用方式也不同，所以需要对不同的选项使用不同的策略进行合并，代码如下所示：

```js
export function mergeOptions(
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  // ...

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

  // ...
}
```

可以看到，通过遍历传入的配置对象，获取到对应的选项后，再从`strats`对象上取出对应的合并策略函数，最后将`parent`上的配置和`child`上的配置通过策略函数进行合并，最终就可以得到合并后的值了。接下来就看看常见的选项是如何进行合并的。

### lifecycle

`lifecycle`生命周期的合并策略，代码如下所示：

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

可以看到，所有生命周期的策略函数都是`mergeHook`，代码如下所示：

```js
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
```

`mergeHook`方法会将`parent`和`child`中的生命周期钩子函数做合并操作，返回一个新的数组，其中`parent`中的钩子函数在前，`child`中的钩子函数在后，最后使用`dedupeHooks`方法过滤掉重复的函数。从这里可以看到，生命周期钩子函数最终会合并成一个数组的形式。

### component、directive、filter

资源相关的合并策略，代码如下所示：

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

可以看到，`component`、`directive`、`filter`的策略函数都是`mergeAssets`，代码如下所示：

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

`mergeAssets`方法首先会以`parentVal`为原型创建一个新的实例，然后把`childVal`合并到这个新实例上。之所以该合并策略使用原型继承的方式，是因为`component`、`directive`、`filter`这些资源相关的选项，在多个实例之间是可以复用的，这样就不用把`parentVal`添加到各个实例上了。

### data、provide

`data`选项的合并策略，代码如下所示：

```js
/* core/util/options.js */
strats.data = function (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    if (childVal && typeof childVal !== 'function') {
      process.env.NODE_ENV !== 'production' && warn(
        'The "data" option should be a function ' +
        'that returns a per-instance value in component ' +
        'definitions.',
        vm
      )

      return parentVal
    }
    return mergeDataOrFn(parentVal, childVal)
  }

  return mergeDataOrFn(parentVal, childVal, vm)
}

export function mergeDataOrFn(
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    // in a Vue.extend merge, both should be functions
    if (!childVal) {
      return parentVal
    }
    if (!parentVal) {
      return childVal
    }
    // when parentVal & childVal are both present,
    // we need to return a function that returns the
    // merged result of both functions... no need to
    // check if parentVal is a function here because
    // it has to be a function to pass previous merges.
    return function mergedDataFn() {
      return mergeData(
        typeof childVal === 'function' ? childVal.call(this, this) : childVal,
        typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
      )
    }
  } else {
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
}
```

可以看到，对于非组件的配置合并，此时的`vm`表示当前`Vue`实例，所以直接调用`mergeDataOrFn`方法，并返回一个新的函数`mergedInstanceDataFn`，这个函数会在`initState`的过程中执行。执行时，首先从`parent`和`child`中取到各自的数据，然后调用`mergeData`方法合并这两项数据，其代码如下所示：

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
      set(to, key, fromVal)
    } else if (
      toVal !== fromVal &&
      isPlainObject(toVal) &&
      isPlainObject(fromVal)
    ) {
      mergeData(toVal, fromVal)
    }
  }
  return to
}
```

可以看到，`mergeData`方法就是一个深度合并的过程，主要是用来将`from`中的数据合并到`to`中，所以首先遍历`from`对象，如果`to`中不存在该属性，就将数据直接添加到`to`中；如果`to`中存在该属性，并且对应的值还是一个对象，就继续调用`mergeData`方法，深度递归该对象，如果对应的值是一个基础类型，那么就不做任何操作，还是以`to`中的值为准，最终可以得到一个以`to`为主的对象，这样就可以把`child`和`parent`中的数据合并起来了。

`provide`选项的合并策略同样使用`mergeDataOrFn`方法，代码如下所示：

```js
/* core/util/options.js */
strats.provide = mergeDataOrFn
```

所以最终的处理结果和`data`类似，也是一个深度合并的结果。

### props、methdos、inject、computed

`props`、`methdos`、`inject`、`computed`的合并策略，代码如下所示：

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

可以看到，这些合并策略很简单，就是简单的浅合并`parentVal`和`childVal`，并且`childVal`中的值可以覆盖`parentVal`中的值。

### 默认策略

对于没有指定策略函数的选项来说，就使用默认策略，代码如下所示：

```js
/* core/util/options.js */
const defaultStrat = function (parentVal: any, childVal: any): any {
  return childVal === undefined
    ? parentVal
    : childVal
}
```

可以看到，默认策略就是如果在`child`中定义了选项，就直接取`child`中定义的值，否则，就取`parent`中定义的值。

在各种选项经过对应的策略函数处理后，将最终的合并结果赋值给`vm.$options`，这样就完成了非组件的配置合并。

那么接下来，就来看看对于组件来说，它是如何合并的。

## 组件的配置合并

其实对于组件来说，配置合并首先会发生在创建组件`VNode`的过程中，在此过程中会调用`Vue.extend`方法，将我们定义的组件配置转换成组件构造器，其代码如下所示：

```js
/* core/global-api/extend.js */
Vue.extend = function (extendOptions: Object): Function {
  extendOptions = extendOptions || {}
  const Super = this
  const SuperId = Super.cid
  const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
  if (cachedCtors[SuperId]) {
    return cachedCtors[SuperId]
  }

  const name = extendOptions.name || Super.options.name
  if (process.env.NODE_ENV !== 'production' && name) {
    validateComponentName(name)
  }

  const Sub = function VueComponent(options) {
    this._init(options)
  }
  Sub.prototype = Object.create(Super.prototype)
  Sub.prototype.constructor = Sub
  Sub.cid = cid++
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

可以看到，`extend`静态方法首先会检查在缓存中是否存在对应的子构造器，如果存在就直接返回，如果不存在，就先创建了一个子构造器`Sub`，然后使用原型继承的方式继承`Super`，在这里也就是`Vue`构造函数。接着调用`mergeOptions`方法，将`Super.options`和我们定义的组件配置选项进行合并，将合并后的结果添加到`Sub.options`属性上，也就是子构造器的`options`选项，合并完成后，处理选项中的`props`、`computed`选项，将它们定义到子构造器的原型上，避免在实例化时，在子实例上多次定义重复逻辑，具体内容在数据响应化部分再详细介绍。然后将`Super`上的静态方法添加到`Sub`上，比如`extend`、`mixin`、`use`、`component`、`directive`、`filter`等，此时，`Sub`构造器就拥有了`Super`构造器的能力，同时将`Sub`自己添加到`Sub.options.components`对象中，这样在组件模板中就可以递归调用自己。最后将构造出的子构造器`Sub`缓存到组件配置对象中，避免重复创建该子构造器。

通过`Vue.extend`创建完子构造器后，此时还没有创建子组件实例，直到当父组件执行`patch`，准备挂载父占位符节点时，才会调用组件的`init`钩子函数，在此方法中又会调用`createComponentInstanceForVnode`方法，这个方法就是创建子组件实例的真正方法，其代码如下所示：

```js
/* core/vdom/create-component.js */
export function createComponentInstanceForVnode(
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
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

可以看到，在`createComponentInstanceForVnode`方法中，首先构建组件配置选项`options`，其中`_isComponent`属性表示当前是组件的配置合并操作，其余的两个`_parentVnode`和`parent`属性分别表示组件的父占位符节点和组件的父`Vue`实例，然后通过`vnode.componentOptions.Ctor`构造函数，也就是上面的`Sub`构造器，创建子组件的实例，最终还是调用上一章节中的`_init`方法，只是这次传入的选项`options`是`Vue`内部构造的，我们在`.vue`文件中定义的组件配置选项，已经在构造`Sub`构造器时合并到`Sub.options`上了，接下来，来看看对于组件的配置合并，`_init`方法是如何工作的，代码如下所示：

```js
/* core/instance/init.js */
Vue.prototype._init = function (options?: Object) {
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  }
}
```

可以看到，创建子组件实例时，会调用`initInternalComponent`方法进行配置合并，其代码如下所示：

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

可以看到，在`initInternalComponent`方法中，首先创建一个继承自`Sub.options`的实例，然后将各种与当前`vm`实例相关`propsData`、`listeners`、`children`等属性添加到该实例上，这样一来，组件构造器上的配置就可以重复使用，避免因创建多个组件实例，调用`mergeOptions`方法而导致的性能损失。

## resolveConstructorOptions

在前两的两小节中，我们已经知道`Vue`是如何合并配置的了，其实在它们的内部，都会执行`resolveConstructorOptions`方法，用来检查构造器上的`options`选项是否是最新值，其代码如下所示：

```js
/* core/instance/init.js */
export function resolveConstructorOptions(Ctor: Class<Component>) {
  let options = Ctor.options
  if (Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
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

function resolveModifiedOptions(Ctor: Class<Component>): ?Object {
  let modified
  const latest = Ctor.options
  const sealed = Ctor.sealedOptions
  for (const key in latest) {
    if (latest[key] !== sealed[key]) {
      if (!modified) modified = {}
      modified[key] = latest[key]
    }
  }
  return modified
}
```

可以看到，在上面的代码中，使用到了`super`、`superOptions`、`extendOptions`、`sealedOptions`这些属性，它们是在`Vue.extend`中设置的：

```js
/* core/global-api/extend.js */
Vue.extend = function (extendOptions: Object): Function {
  Sub['super'] = Super

  Sub.superOptions = Super.options
  Sub.extendOptions = extendOptions
  Sub.sealedOptions = extend({}, Sub.options)
}
```

如上所示，`super`表示父构造器，`superOptions`表示父构造器的配置对象，`extendOptions`表示组件的配置对象，`sealedOptions`表示合并后的配置对象的拷贝，所以在`resolveConstructorOptions`方法中，如果检测到组件上的`superOptions`与父构造器的`options`不相等，则说明在创建完子构造器后，父构造器的配置选项进行过更新，就需要通过`resolveModifiedOptions`方法，找到新旧父`options`之间的区别后，再重新使用`mergeOptions`方法构造新的子`options`，所以使用`resolveConstructorOptions`方法就可以保证在创建实例时，构造器上的配置`options`是最新的了。

## 总结

在`Vue`中，每个实例的`$options`不仅仅来自于我们编写的配置，它还会将组件构造器上的配置，通过不同的策略函数，进行合并，从而得到最终的配置，同时对于非组件的配置合并和组件的配置合并来说，它们的处理流程是不相同的，对于组件的配置合并来说，它只会执行一次`mergeOptions`操作，每个组件自己的数据都会在构建组件占位符节点时，进行构建，然后在实例化组件时传入。
