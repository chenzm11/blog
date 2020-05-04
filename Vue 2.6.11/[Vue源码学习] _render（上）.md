# [Vue源码学习] _render（上）

## 前言

从上一章节中，我们知道，在调用`$mount`方法进行挂载的过程中，会执行`updateComponent`方法，其代码如下所示：

```js
/* core/instance/lifecycle.js */
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  // ...
  updateComponent = () => {
      vm._update(vm._render(), hydrating)
  }
  // ...
}
```

可以看到，在`updateComponent`方法中，第一步是调用`vm._render`方法，用来生成`VNode`；第二步是调用`vm._update`方法，通过`VNode`渲染成最终的`DOM`。那么接下来，就来看看`_render`方法是如何生成`VNode`的。

## _render

`_render`方法是在引入`Vue`时，添加到`Vue`原型上的，其代码如下所示：

```js
/* core/instance/render.js */
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const { render, _parentVnode } = vm.$options

  if (_parentVnode) {
    vm.$scopedSlots = normalizeScopedSlots(
      _parentVnode.data.scopedSlots,
      vm.$slots,
      vm.$scopedSlots
    )
  }

  // set parent vnode. this allows render functions to have access
  // to the data on the placeholder node.
  vm.$vnode = _parentVnode
  // render self
  let vnode
  try {
    // There's no need to maintain a stack because all render fns are called
    // separately from one another. Nested component's render fns are called
    // when parent component is patched.
    currentRenderingInstance = vm
    vnode = render.call(vm._renderProxy, vm.$createElement)
  } catch (e) {
    handleError(e, vm, `render`)
    // return error render result,
    // or previous vnode to prevent render error causing blank component
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production' && vm.$options.renderError) {
      try {
        vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
      } catch (e) {
        handleError(e, vm, `renderError`)
        vnode = vm._vnode
      }
    } else {
      vnode = vm._vnode
    }
  } finally {
    currentRenderingInstance = null
  }
  // if the returned array contains only a single node, allow it
  if (Array.isArray(vnode) && vnode.length === 1) {
    vnode = vnode[0]
  }
  // return empty vnode in case the render function errored out
  if (!(vnode instanceof VNode)) {
    if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
      warn(
        'Multiple root nodes returned from render function. Render function ' +
        'should return a single root node.',
        vm
      )
    }
    vnode = createEmptyVNode()
  }
  // set parent
  vnode.parent = _parentVnode
  return vnode
}
```

可以看到，在`_render`方法中，首先从`$options`上取出渲染函数`render`、父占位符节点`_parentVnode`，接着判断如果存在父占位符节点，则通过调用`normalizeScopedSlots`方法，解析作用域插槽的相关内容，关于这部分内容，会在之后的章节中详细介绍；然后将父占位符节点`_parentVnode`赋值给`vm.$vnode`，需要注意的是，对于根组件来说，由于不存在父占位符节点，所以此时`rootvm.$vnode`为`undefined`，所以根组件会在`$mount`方法的最后会调用`mounted`钩子函数，而子组件则不会调用。

接着通过变量`currentRenderingInstance`保存当前正在处理的`Vue`实例，这样在接下来的`render`过程中，就可以访问到当前实例了，需要注意的是，这里没有使用栈来保存`Vue`实例，是因为在同一时间，有且仅有一个`Vue`实例会完整的调用`render`方法，与下一章将要介绍的`_update`方法不同；然后就是调用配置选项中的`render`渲染函数，用来生成`VNode`，这里的`render`函数可以是经过`vue-loader`处理过的`template`模板，也可以是手写`render`函数，里面具体的内容根据编写的模板的不同而不同；在生成好`VNode`之后，将`currentRenderingInstance`置空，同时将父占位符节点`_parentVnode`赋值给`vnode.parent`，最后返回生成的`VNode`节点。

那么接下来，来看看在`render`函数内部，是如何生成`VNode`的。

## createElement

在`Vue`中，可以通过`vm._c`或`vm.$createElement`方法创建`VNode`节点，其定义如下所示：

```js
/* core/instance/render.js */
export function initRender(vm: Component) {
  // ...
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
  // ...
}
```

其中，经过`vue-loader`生成的渲染函数，其内部使用的是`vm._c`，而手写渲染函数时，传入的第一个参数是`vm.$createElement`。可以看到，这两个方法内部都是调用`createElement`方法，只是最后一个参数不同。

其实简单点来说，`createElement`就是用来创建`VNode`的方法，而对于每个`VNode`节点，最关键的无非是三个东西：

1. `tag`：标签名，表示当前`VNode`是何种标签元素，比如`div`、`p`，也可以表示组件。

2. `data`：节点数据，表示当前`VNode`上所有与之相关的数据，比如`attrs`、`on`等。

3. `children`：子节点，表示当前`VNode`的所有子节点。

那么接下来，我们就详细看看`createElement`方法的逻辑，其代码如下所示：

```js
/* core/vdom/create-element.js */
export function createElement(
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}

export function _createElement(
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== 'production' && warn(
      `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
      'Always create fresh vnode data objects in each render!',
      context
    )
    return createEmptyVNode()
  }
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // warn against non-primitive key
  if (process.env.NODE_ENV !== 'production' &&
    isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    if (!__WEEX__ || !('@binding' in data.key)) {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      )
    }
  }
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      if (process.env.NODE_ENV !== 'production' && isDef(data) && isDef(data.nativeOn)) {
        warn(
          `The .native modifier for v-on is only valid on components but it was used on <${tag}>.`,
          context
        )
      }
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}
```

可以看到，`createElement`就是一个包装函数，在格式化传入的参数后，就调用真正的创建`VNode`的方法`_createElement`。在该方法中，首先取得节点的标签名`tag`；然后通过`normalizationType`参数的类型，使用不同的方法格式化子节点，需要注意的是，此时子节点可能已经是`VNode`节点了，这里的逻辑只是确保`children`是一个数组，方便之后的逻辑处理。在得到`tag`、`data`、`children`后，就可以根据这些参数生成真正的`VNode`节点了，首先判断`tag`是否是字符串类型，因为`tag`也可能指向一个组件名，所以首先通过`isReservedTag`方法判断`tag`标签名是否是平台内置的标签元素，其代码如下所示：

```js
/* platforms/web/util/element.js */
export const isReservedTag = (tag: string): ?boolean => {
  return isHTMLTag(tag) || isSVG(tag)
}

export const isHTMLTag = makeMap(
  'html,body,base,head,link,meta,style,title,' +
  'address,article,aside,footer,header,h1,h2,h3,h4,h5,h6,hgroup,nav,section,' +
  'div,dd,dl,dt,figcaption,figure,picture,hr,img,li,main,ol,p,pre,ul,' +
  'a,b,abbr,bdi,bdo,br,cite,code,data,dfn,em,i,kbd,mark,q,rp,rt,rtc,ruby,' +
  's,samp,small,span,strong,sub,sup,time,u,var,wbr,area,audio,map,track,video,' +
  'embed,object,param,source,canvas,script,noscript,del,ins,' +
  'caption,col,colgroup,table,thead,tbody,td,th,tr,' +
  'button,datalist,fieldset,form,input,label,legend,meter,optgroup,option,' +
  'output,progress,select,textarea,' +
  'details,dialog,menu,menuitem,summary,' +
  'content,element,shadow,template,blockquote,iframe,tfoot'
)
```

如果是内置标签，就直接通过`VNode`构造器创建一个元素`VNode`节点；如果不是内置标签，就通过`resolveAsset`方法，尝试通过`tag`组件名从`$options.components`中找到当前组件的定义：

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

可以看到，组件支持连字符、驼峰、大写三种书写方式，如果能够找到组件的配置对象，那么就使用`createComponent`方法创建组件`VNode`，这部分内容会在下一章节中详细介绍；最后如果既不是内置标签，也没有找到该组件的定义，就会通过标签名创建一个`unknown VNode`节点。以上逻辑是当`tag`是一个字符串时的处理流程，而如果`tag`是一个对象，那么就会直接通过`createComponent`方法创建组件`VNode`。

不管是何种逻辑，`_createElement`方法最终都会返回一个`VNode`节点，需要注意的是，这里只是生成元素`VNode`和组件`VNode`，而其他类型的`VNode`节点，由于不存在`data`或`children`，只需要简单创建即可，比如对于普通文本，则是通过`createTextVNode`方法直接创建的。

最终，`_render`方法就返回了当前`Vue`实例的根`VNode`节点，而通过`VNode.children`属性，又能访问到它的子节点，这样就构成了一棵虚拟节点树，可以从根`VNode`节点，逐级遍历所有的`VNode`节点。

## 总结

在`_render`方法中，`Vue`会使用`_createElement`方法根据`tag`创建不同类型的`VNode`节点，而每个`VNode`节点最重要的属性就是`tag`、`data`、`children`，通过这些属性，就可以描述一个节点的详细信息，进而渲染对应的真实`DOM`节点。
