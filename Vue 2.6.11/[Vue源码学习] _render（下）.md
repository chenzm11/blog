# [Vue源码学习] _render（下）

## 前言

从上一章节中，我们知道，通过调用`createElement`方法，可以直接创建元素`VNode`，而对于组件来说，则是通过`createComponent`方法创建组件`VNode`，那么接下来，就来看看该方法是如何创建组件节点的。

## createComponent

`createComponent`方法的代码如下所示：

```js
/* core/vdom/create-component.js */
export function createComponent(
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }

  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // if at this stage it's not a constructor or an async component factory,
  // reject.
  if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
  }

  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }

  data = data || {}

  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  resolveConstructorOptions(Ctor)

  // transform component v-model data into props & events
  if (isDef(data.model)) {
    transformModel(Ctor.options, data)
  }

  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)

  // functional component
  if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children)
  }

  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  const listeners = data.on
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn

  if (isTrue(Ctor.options.abstract)) {
    // abstract components do not keep anything
    // other than props & listeners & slot

    // work around flow
    const slot = data.slot
    data = {}
    if (slot) {
      data.slot = slot
    }
  }

  // install component management hooks onto the placeholder node
  installComponentHooks(data)

  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  return vnode
}
```

通过上一章的分析，我们知道这里的参数`Ctor`是组件的配置对象，而`baseCtor`指向的是`Vue`构造函数，所以接下来就是调用`Vue.extend(Ctor)`方法，通过组件配置对象，生成组件构造器，关于`extend`方法的具体实现方式，在**配置合并**章节中已经进行过详细介绍；在得到组件构造器后，接下来就是对异步组件的处理，这部分内容会在之后的章节中再单独介绍；然后就是处理组件`data`的内容了，首先处理了`model`选项，然后调用`extractPropsFromVNodeData`方法从`data.props`、`data.attrs`中提取出`propsData`，处理原生`DOM`事件和组件自定义事件等逻辑；接着就是调用`installComponentHooks`方法，注册组件钩子，代码如下所示：

```js
/* core/vdom/create-component.js */
function installComponentHooks(data: VNodeData) {
  const hooks = data.hook || (data.hook = {})
  for (let i = 0; i < hooksToMerge.length; i++) {
    const key = hooksToMerge[i]
    const existing = hooks[key]
    const toMerge = componentVNodeHooks[key]
    if (existing !== toMerge && !(existing && existing._merged)) {
      hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge
    }
  }
}

const hooksToMerge = Object.keys(componentVNodeHooks)

// inline hooks to be invoked on component VNodes during patch
const componentVNodeHooks = {
  init(vnode: VNodeWithData, hydrating: boolean): ?boolean {
    // ...
  },

  prepatch(oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    // ...
  },

  insert(vnode: MountedComponentVNode) {
    // ...
  },

  destroy(vnode: MountedComponentVNode) {
    // ...
  }
}
```

可以看到，这部分逻辑主要就是在组件的`data.hook`上注册`init`、`prepatch`、`insert`、`destroy`钩子函数，这样组件在之后的初始化，更新，卸载等过程中，就可以通过这些钩子函数，进行统一处理了；添加完钩子函数后，接着调用`VNode`构造函数，创建组件`VNode`节点，需要注意的是，对于组件`VNode`来说，是没有`children`子节点的，组件的子节点其实就是`slot`插槽，所以将这部分内容放到了组件`VNode`的`componentOptions`中。最终，将生成的组件`VNode`对象返回，这样就完成了组件`VNode`的创建过程。

## 总结

通过`createComponent`方法，就可以完成对组件`VNode`的初始化工作，同时将与之相关的数据，比如组件构造器`Ctor`、属性`propsData`、自定义事件`listeners`、插槽`children`等，放在`componentOptions`中并绑定到该组件`VNode`上，同时给该组件各个阶段注册了钩子函数，以便`Vue`能够统一控制组件的创建、更新、销毁过程。
