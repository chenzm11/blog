# [Vue源码学习] _render（下）

## 前言

在上一章节中，我们可以通过`createElement`方法，创建普通元素`VNode`，那么在本章节中，我们就来看看`Vue`是如何创建组件`VNode`的。

## _createElement

在`_createElement`方法中，有两种创建组件`VNode`的方式：

```js
/* core/vdom/create-element.js */
export function _createElement(
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // ...
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
}
```

1. `tag`是一个字符串，并且不是平台内置标签，就会通过`resolveAsset`方法，尝试通过`tag`从`$options.components`中找到组件的定义：

    ```js
    /* core/util/options.js */
    export function resolveAsset(
      options: Object,
      type: string,
      id: string,
      warnMissing?: boolean
    ): any {
      /* istanbul ignore if */
      if (typeof id !== 'string') {
        return
      }
      const assets = options[type]
      // check local registration variations first
      if (hasOwn(assets, id)) return assets[id]
      const camelizedId = camelize(id)
      if (hasOwn(assets, camelizedId)) return assets[camelizedId]
      const PascalCaseId = capitalize(camelizedId)
      if (hasOwn(assets, PascalCaseId)) return assets[PascalCaseId]
      // fallback to prototype chain
      const res = assets[id] || assets[camelizedId] || assets[PascalCaseId]
      if (process.env.NODE_ENV !== 'production' && warnMissing && !res) {
        warn(
          'Failed to resolve ' + type.slice(0, -1) + ': ' + id,
          options
        )
      }
      return res
    }
    ```

    如果找到了组件的定义，就会调用`createComponent`方法，创建组件`VNode`。

2. `tag`是一个对象，直接调用`createComponent`方法，创建组件`VNode`。

那么接下来，我们就来看看`createComponent`方法的具体实现。

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

  // 通过组件配置对象生成组件构造器
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // 异步组件
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

  // 在Super.options更新时，重新调用mergeOptions方法，生成Sub.options
  resolveConstructorOptions(Ctor)

  // 处理v-model指令
  if (isDef(data.model)) {
    transformModel(Ctor.options, data)
  }

  // 从data.attrs和data.props中提取组件prop
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)

  // 函数组件
  if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children)
  }

  // 保存组件自定义事件后，交换native事件
  const listeners = data.on
  data.on = data.nativeOn

  // 抽象组件
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

  // 注册组件通用钩子函数
  installComponentHooks(data)

  // 生成组件VNode
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

在`createComponent`方法中，首先会调用`extend`方法，通过组件配置对象生成组件构造器，这部分的内容在**配置合并**章节中已经进行过详细介绍；在看接下来的逻辑之前，首先需要明确一件事情，这里创建的组件节点，是在父组件执行`render`的过程中生成的，它只是一个单纯的组件占位符节点，此时不会解析该组件内部的渲染内容，并且对于这个父占位符节点来说，到真正需要实例化子组件的时候，要为子实例提供`propsData`、`listeners`、`children`数据，所以接下来的逻辑，主要就是构建这几项数据，所以在`createComponent`方法中，首先会从`data`中提取`propsData`和`listeners`，然后调用`installComponentHooks`方法，向占位符节点注册通用的钩子函数：

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

可以看到，这部分逻辑主要就是在组件的父占位符节点的`data.hook`上注册`init`、`prepatch`、`insert`、`destroy`钩子函数，这样组件在之后的初始化，更新，卸载等过程中，就可以通过这些钩子函数，做相应的操作，注册完钩子函数后，就调用`VNode`构造函数，创建组件`VNode`，需要注意的是，之后传给子实例`propsData`、`listeners`、`children`，是作为`componentOptions`保存在组件`VNode`中的，最终，将创建好的节点返回。

## 总结

通过`createComponent`方法，就可以创建组件`VNode`，同时将与之相关的数据，比如组件构造器`Ctor`、属性`propsData`、自定义事件`listeners`、插槽`children`等，保存在`componentOptions`中，最后还给该组件各个阶段注册了钩子函数，以便`Vue`能够统一控制组件的创建、更新、销毁过程。
