# [Vue源码学习] _render（上）

## 前言

从上一章节中我们知道，在调用`$mount`进行挂载的过程中，会执行`updateComponent`方法：

```js
/* core/instance/lifecycle.js */
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }
}
```

可以看到，在`updateComponent`方法中，首先会调用`_render`方法，生成组件对应的`VNode`，然后调用`_update`方法，根据`VNode`渲染成真实的`DOM`。那么接下来，我们就来看看`_render`方法是如何生成`VNode`的。

## _render

`_render`是在引入`Vue`时添加到`Vue.prototype`上的，代码如下所示：

```js
/* core/instance/render.js */
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const { render, _parentVnode } = vm.$options

  // 解析作用域插槽
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
    // 根据渲染函数，生成VNode
    vnode = render.call(vm._renderProxy, vm.$createElement)
  } catch (e) {
    // ...
  } finally {
    currentRenderingInstance = null
  }
  // if the returned array contains only a single node, allow it
  if (Array.isArray(vnode) && vnode.length === 1) {
    vnode = vnode[0]
  }
  // return empty vnode in case the render function errored out
  if (!(vnode instanceof VNode)) {
    // ...
    vnode = createEmptyVNode()
  }
  // set parent
  vnode.parent = _parentVnode
  return vnode
}
```

可以看到，在`_render`方法中，首先从`$options`上取出渲染函数`render`、父占位符节点`_parentVnode`，对于子组件来说，此时还会调用`normalizeScopedSlots`，用于解析作用域插槽，然后将`_parentVnode`赋值给实例的`$vnode`，从这里可以看出，由于根组件不存在父占位符节点，所以它的`$vnode`为`undefined`，当根组件完成挂载后，在`$mount`的最后会调用`mounted`钩子函数，而子组件则会跳过这段逻辑：

```js
/* core/instance/lifecycle.js */
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
}
```

接着就会调用`render`渲染函数，它是`_render`的核心逻辑，除了生成`VNode`外，`Vue`还在此过程中完成了对依赖的收集。这里的`render`函数可以是经过`vue-loader`处理过的`template`模板，也可以是用户手写的`render`函数，总之，`render`的返回结果就是当前组件的渲染根`VNode`，然后通过`parent`属性，构建渲染`VNode`与占位符`VNode`之间的父子关系。

那么接下来，我们就来看看对于单个节点，`Vue`是如何生成`VNode`的。

## createElement

`Vue`提供了两个方法生成`VNode`，一个是`_c`，一个是`$createElement`，通过`vue-loader`生成的渲染函数，其内部使用的是`vm._c`，手写渲染函数时，传入的第一个参数是`vm.$createElement`：

```js
/* core/instance/render.js */
export function initRender(vm: Component) {
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
}
```

可以看到，这两个方法内部都是调用`createElement`方法，只是最后一个参数不同。

其实简单点来说，`createElement`就是用来创建`VNode`，而对于每个`VNode`节点，最关键的无非是三样东西：

1. `tag`：标签名，表示当前`VNode`是何种标签元素，比如`div`、`p`，也可以表示组件。

2. `data`：节点数据，表示当前`VNode`上所有与之相关的数据，比如`attrs`、`on`等。

3. `children`：子节点，表示当前`VNode`的所有子节点。

那么接下来，我们就详细看看`createElement`方法的实现：

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
  // ...
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // ...
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  // 规范化子节点
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  // 根据tag，生成普通节点或组件节点
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
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

可以看到，`createElement`就是对`_createElement`方法的包装，在`_createElement`方法中，首先根据`normalizationType`，调用`normalizeChildren`或`simpleNormalizeChildren`方法，规范化子节点，其实内部就是用来合并相邻的文本节点，将嵌套子节点展平的逻辑，处理完成后，`children`是一个`VNode`类型的一维数组。

接下来，就是根据`tag`的类型生成不同种类的`VNode`节点，我们首先来看看对于普通元素节点，`Vue`是如何创建`VNode`的。对于普通元素节点来说，其`tag`肯定是一个字符串，所以`typeof tag === 'string'`为`true`，接着通过`isReservedTag`判断`tag`是否是平台内置的标签元素，如果是的话，则说明该节点是普通元素节点，就直接调用`VNode`构造函数，创建元素节点对应的`VNode`，`isReservedTag`方法的代码如下所示：

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

最后，就可以通过`tag`、`data`、`children`，创建一个`VNode`，它所包含的信息就会告诉`Vue`页面上需要渲染什么样的节点以及其子节点。

通过上面的分析，我们已经知道`tag`和`children`分别代表标签名和子节点，那么`data`中又应该包含哪些信息呢？我们可以从`VNodeData`中找到：

```js
export interface VNodeData {
  key?: string | number;
  // 普通命名插槽
  slot?: string;
  // 作用域插槽(在父组件占位符VNode中存在)
  scopedSlots?: { [key: string]: ScopedSlot | undefined };
  ref?: string;
  refInFor?: boolean;
  tag?: string;
  staticClass?: string;
  class?: any;
  staticStyle?: { [key: string]: any };
  style?: string | object[] | object;
  // 组件的prop
  props?: { [key: string]: any };
  // html attribute 通过el.setAttribute设置
  attrs?: { [key: string]: any };
  // DOM property 通过el[prop]=xxx设置
  domProps?: { [key: string]: any };
  // VNode的hook
  hook?: { [key: string]: Function };
  // html原生事件 + 组件自定义事件
  on?: { [key: string]: Function | Function[] };
  // 定义在组件上的原生事件
  nativeOn?: { [key: string]: Function | Function[] };
  transition?: object;
  show?: boolean;
  inlineTemplate?: {
    render: Function;
    staticRenderFns: Function[];
  };
  // 自定义指令
  directives?: VNodeDirective[];
  keepAlive?: boolean;
}
```

## 总结

通过`createElement`方法，可以生成对应的`VNode`节点，对于一个普通的元素节点来说，只需要给定`tag`、`data`、`children`，就可以完整的描述对应的真实节点。
