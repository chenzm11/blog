# [Vue源码学习] props

## 前言

从之前的章节中，我们知道`Vue`是如何将普通数据转换为响应式数据，但是组件除了拥有自身的数据外，还可以接收来自父组件中传入的数据，那么在本章节中，我们就来看看`Vue`是如何处理来自外部的数据。

## propsData

由于`props`选项是用来接收来自父组件的数据，所以首先得从父组件构造`propsData`说起。在创建组件的父占位符的过程中，会调用`Vue.extend`方法，构造子组件构造器，在此过程中，会调用`normalizeProps`方法，规范化组件的`props`选项，代码如下所示：

```js
/* core/util/options.js */
function normalizeProps(options: Object, vm: ?Component) {
  const props = options.props
  if (!props) return
  const res = {}
  let i, val, name
  if (Array.isArray(props)) {
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val)
        res[name] = { type: null }
      } else if (process.env.NODE_ENV !== 'production') {
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  options.props = res
}
```

可以看到，由于`props`选项支持数组、对象等多种书写格式，所以需要使用`normalizeProps`方法，使每个数据的配置都规范化为统一的格式，从而方便之后的解析。在规范化完成后，在`Vue.extend`方法中，还会调用`initProps`方法，将`props`代理到组件的原型上，因为这部分数据是可以共享的，其代码如下所示：

```js
/* core/global-api/extend.js */
Vue.extend = function (extendOptions: Object): Function {
  // ...
  if (Sub.options.props) {
    initProps(Sub)
  }
  // ...
}

function initProps(Comp) {
  const props = Comp.options.props
  for (const key in props) {
    proxy(Comp.prototype, `_props`, key)
  }
}
```

可以看到，在`initProps`方法中，主要就是将数据代理到组件原型上的`_props`对象中。处理完`props`选项后，就调用`extractPropsFromVNodeData`方法来构建`propsData`，代码如下所示：

```js
/* core/vdom/create-component.js */
export function createComponent(
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  // ...
  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)
  // ...
}

/* core/vdom/helpers/extract-props.js */
export function extractPropsFromVNodeData(
  data: VNodeData,
  Ctor: Class<Component>,
  tag?: string
): ?Object {
  // we are only extracting raw values here.
  // validation and default values are handled in the child
  // component itself.
  const propOptions = Ctor.options.props
  if (isUndef(propOptions)) {
    return
  }
  const res = {}
  const { attrs, props } = data
  if (isDef(attrs) || isDef(props)) {
    for (const key in propOptions) {
      const altKey = hyphenate(key)
      if (process.env.NODE_ENV !== 'production') {
        const keyInLowerCase = key.toLowerCase()
        if (
          key !== keyInLowerCase &&
          attrs && hasOwn(attrs, keyInLowerCase)
        ) {
          tip(
            `Prop "${keyInLowerCase}" is passed to component ` +
            `${formatComponentName(tag || Ctor)}, but the declared prop name is` +
            ` "${key}". ` +
            `Note that HTML attributes are case-insensitive and camelCased ` +
            `props need to use their kebab-case equivalents when using in-DOM ` +
            `templates. You should probably use "${altKey}" instead of "${key}".`
          )
        }
      }
      checkProp(res, props, key, altKey, true) ||
        checkProp(res, attrs, key, altKey, false)
    }
  }
  return res
}
```

可以看到，在`extractPropsFromVNodeData`方法中，首先从组件配置中提取刚刚处理过的`props`选项，并赋值给`propOptions`，然后从`VNodeData`中提取`attrs`和`props`，接着开始遍历`propOptions`，处理每一个数据，首先调用`hyphenate`方法，将属性名处理成连字符的形式，然后使用`checkProp`方法尝试从`attrs`或`props`中提取数据，代码如下所示：

```js
/* core/vdom/helpers/extract-props.js */
function checkProp(
  res: Object,
  hash: ?Object,
  key: string,
  altKey: string,
  preserve: boolean
): boolean {
  if (isDef(hash)) {
    if (hasOwn(hash, key)) {
      res[key] = hash[key]
      if (!preserve) {
        delete hash[key]
      }
      return true
    } else if (hasOwn(hash, altKey)) {
      res[key] = hash[altKey]
      if (!preserve) {
        delete hash[altKey]
      }
      return true
    }
  }
  return false
}
```

可以看到，首先尝试从`props`中提取数据，否则尝试从`attrs`中提取数据，如果数据存在于`attrs`中，还会将数据从`attrs`中删除。

经过`extractPropsFromVNodeData`方法处理后，就可以得到从父组件中传到子组件的数据`propsData`，在`createComponent`方法的最后，会将`propsData`放到`VNode.componentOptions`中。

从前面的章节中，我们知道，在父组件`patch`的过程中，会创建子组件的实例，同时也会将组件的父占位符`VNode`当作配置选项传入，所以在子组件调用`initInternalComponent`方法进行配置合并的过程中，就可以从父占位符`VNode`中提取`propsData`。那么接下来，就来看看子组件是如何处理该数据的。

## initProps

在初始化子组件，得到`propsData`后，会调用`initState`方法，在该方法中，又会调用`initProps`方法，这个方法就是用来处理`propsData`的，代码如下所示：

```js
/* core/instance/state.js */
export function initState(vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  // ...
}

function initProps(vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      const hyphenatedKey = hyphenate(key)
      if (isReservedAttribute(hyphenatedKey) ||
        config.isReservedAttr(hyphenatedKey)) {
        warn(
          `"${hyphenatedKey}" is a reserved attribute and cannot be used as component prop.`,
          vm
        )
      }
      defineReactive(props, key, value, () => {
        if (!isRoot && !isUpdatingChildComponent) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

可以看到，在`initProps`方法中，对于非根组件实例，会调用`toggleObserving`方法，阻止嵌套数据的响应式化，然后遍历组件的`props`选项，对每个数据使用`validateProp`方法，该方法是用来检测数据是否满足配置中的`type`、`required`、`validator`选项，在不满足时会提示警告，如果有`default`选项的话，在没有传入数据时返回默认值，`validateProp`方法最终会返回对应的数据。

在得到数据之后，就是调用`defineReactive`方法，将该数据转换为响应式数据，需要注意的是，由于前面调用了`toggleObserving(false)`方法，所以不会对嵌套数据进行响应式化，只有最外层数据才会经过响应式处理，然后判断如果该属性没有定义在`Vue`实例上，就使用`proxy`方法，将`_props`上的数据代理到`Vue`实例上。最后就是调用`toggleObserving(true)`方法，恢复标志位。

经过`initProps`方法处理后，已经将父组件传入的数据，经过响应式处理后，代理到子组件实例上了。但是当在子组件中使用最外层数据时，父组件中对应的数据是无法感知的，也就是说，父组件中该数据对应的`dep`集合中是不包含子组件的渲染`Watcher`的(嵌套数据除外)。那接下来，就来看看在父组件中修改数据时，子组件是如何收到最新的数据，并进行更新的。

## update

首先需要明确的是，对于`props`选项中的最外层数据来说，它在父组件对应的`dep`中只会添加父组件的渲染`Watcher`，所以当数据在父组件中进行修改时，最开始只会将父组件的渲染`Watcher`添加到更新列表`queue`中，而不会添加子组件的渲染`Watcher`，然后在下一帧执行`flushSchedulerQueue`方法，执行父组件的重新渲染。在执行`patch`的过程中，会进行对比更新的逻辑，当开始对比两个子组件的占位符`VNode`时，由于是相同的组件`VNode`，所以同样会调用`patchVnode`方法，在该方法中，会执行组件的`prepatch`钩子函数，更新子组件，代码如下所示：

```js
/* core/vdom/create-component.js */
prepatch(oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
  const options = vnode.componentOptions
  const child = vnode.componentInstance = oldVnode.componentInstance
  updateChildComponent(
    child,
    options.propsData, // updated props
    options.listeners, // updated listeners
    vnode, // new parent vnode
    options.children // new children
  )
}
```

可以看到，在`prepatch`钩子函数中，这里的`options.propsData`就是父组件更新后提取出来的`props`数据，那继续来看`updateChildComponent`方法，代码如下所示：

```js
/* core/instance/lifecycle.js */
export function updateChildComponent(
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
  // ...
  // update props
  if (propsData && vm.$options.props) {
    toggleObserving(false)
    const props = vm._props
    const propKeys = vm.$options._propKeys || []
    for (let i = 0; i < propKeys.length; i++) {
      const key = propKeys[i]
      const propOptions: any = vm.$options.props // wtf flow?
      props[key] = validateProp(key, propOptions, propsData, vm)
    }
    toggleObserving(true)
    // keep a copy of raw propsData
    vm.$options.propsData = propsData
  }
  // ...
}
```

可以看到，在`updateChildComponent`方法中，通过遍历在`initProps`方法中绑定的属性名`_propKeys`，来处理数据，还是同样调用`validateProp`方法检测并取得最新的数值，由于在组件初始化时，`props`中的数据已经定义了响应式，所以这次重新赋值，就会触发该数据对应的`set`访问器，如果检测到数据发生变化，就会将子组件的渲染`Watcher`动态的添加到更新列表`queue`中，当父组件完成更新后，就会触发子组件的重新渲染，最终，父子组件就都得到了更新。

## 总结

`Vue`会根据`props`选项，在创建父占位符节点的时候构建`propsData`，然后在实例化子组件时，将`propsData`进行响应式处理，当数据在父组件中进行更新时，会在对比新旧组件`VNode`的过程中，调用`prepatch`钩子函数，对子组件中的数据进行更新，从而触发子组件的重新渲染。
