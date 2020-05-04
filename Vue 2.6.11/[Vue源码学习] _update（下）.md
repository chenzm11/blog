# [Vue源码学习] _update（下）

## 前言

从上一章节中，我们知道，通过`createElm`方法，`Vue`可以很方便的根据`VNode`创建各种类型的`DOM`节点，而对于组件`VNode`来说，是通过调用`createComponent`方法进行创建的，那么接下来，就来看看该方法是如何渲染组件节点的。

## createComponent

`createComponent`方法的代码如下所示：

```js
/* core/vdom/patch.js */
function createComponent(vnode, insertedVnodeQueue, parentElm, refElm) {
  let i = vnode.data
  if (isDef(i)) {
    const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
    if (isDef(i = i.hook) && isDef(i = i.init)) {
      i(vnode, false /* hydrating */)
    }
    // after calling the init hook, if the vnode is a child component
    // it should've created a child instance and mounted it. the child
    // component also has set the placeholder vnode's elm.
    // in that case we can just return the element and be done.
    if (isDef(vnode.componentInstance)) {
      initComponent(vnode, insertedVnodeQueue)
      insert(parentElm, vnode.elm, refElm)
      if (isTrue(isReactivated)) {
        reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
      }
      return true
    }
  }
}
```

可以看到，`createComponent`方法的逻辑还是很清晰的，这里的`vnode`就是组件的父占位符`VNode`，也就是说，在完成子组件的渲染后，会将整个子组件挂载到此处。该方法首先从`vnode.data.hook`中取出`init`钩子函数，`init`钩子是在创建组件`VNode`时，添加到`hook`上的，其代码如下所示：

```js
/* core/vdom/create-component.js */
init(vnode: VNodeWithData, hydrating: boolean): ?boolean {
  if (
    vnode.componentInstance &&
    !vnode.componentInstance._isDestroyed &&
    vnode.data.keepAlive
  ) {
    // kept-alive components, treat as a patch
    const mountedNode: any = vnode // work around flow
    componentVNodeHooks.prepatch(mountedNode, mountedNode)
  } else {
    const child = vnode.componentInstance = createComponentInstanceForVnode(
      vnode,
      activeInstance
    )
    child.$mount(hydrating ? vnode.elm : undefined, hydrating)
  }
}
```

在`init`方法中，对于首次创建，首先会调用`createComponentInstanceForVnode`方法，创建子组件实例，其代码如下所示：

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

可以看到，在`createComponentInstanceForVnode`方法中，`Vue`在内部首先构建了一个配置对象，然后调用子组件的构造器，从而完成子组件的初始化过程，其中，配置对象中的`_parentVnode`表示子组件的父占位符`VNode`，它的`componentOptions`属性中包含了当前组件的详细信息，`parent`表示子组件的父实例，也就是之后子实例的`$parent`。最后，通过`vnode.componentOptions.Ctor`子构造器，创建子组件的实例，在其内部，同样是调用`_init`方法，所以子组件的创建过程和根组件的创建过程是类似的，其中主要的区别是子组件的配置对象是`Vue`内部构建的，并且其`_isComponent`属性为`true`，所以在配置合并环节，调用的是`initInternalComponent`方法，而不是`mergeOptions`方法，这部分内容在**配置合并**章节中已经进行过详细介绍；另一个区别是，子组件的配置对象中不存在`el`属性，所以在`_init`方法的最后，不会调用`$mount`方法进行挂载。

回到`init`钩子函数中，通过调用`createComponentInstanceForVnode`方法，创建好子实例后，`Vue`会手动调用`child.$mount`方法，挂载子组件，需要注意的是，对于非服务端渲染，此时的挂载元素为空，除此之外，其他逻辑与之前的`$mount`挂载逻辑相同，也是创建子组件的渲染`Watcher`，然后调用`_render`方法生成子组件的`VNode`，接着调用`_update`方法根据子组件`VNode`渲染成真实的`DOM`节点。

调用完`init`钩子函数后，此时调用栈已经回到了父实例中，此时的`vnode.componentInstance`是刚刚创建的子组件实例，所以接着调用`initComponent`方法，其代码如下所示：

```js
/* core/vdom/patch.js */
function initComponent(vnode, insertedVnodeQueue) {
  if (isDef(vnode.data.pendingInsert)) {
    insertedVnodeQueue.push.apply(insertedVnodeQueue, vnode.data.pendingInsert)
    vnode.data.pendingInsert = null
  }
  vnode.elm = vnode.componentInstance.$el
  if (isPatchable(vnode)) {
    invokeCreateHooks(vnode, insertedVnodeQueue)
    setScope(vnode)
  } else {
    // empty component root.
    // skip all element-related modules except for ref (#3455)
    registerRef(vnode)
    // make sure to invoke the insert hook
    insertedVnodeQueue.push(vnode)
  }
}
```

可以看到，在`initComponent`方法中，首先尝试从`vnode`中提取`pendingInsert`，在首次渲染时，`pendingInsert`里面包含了子组件内部创建的所有子组件实例，并将其添加到当前的`insertedVnodeQueue`中，通过层层的提取，所以在最后可以在同一时间调用`insert`钩子函数；然后将子组件的根`DOM`节点赋值给父占位符的`elm`上；接下来判断子该组件是否是可`patchable`的，如果可挂载，那么就需要继续调用`invokeCreateHooks`方法，将父占位符`vnode`上的数据更新到子组件的根节点上，代码如下所示：

```js
/* core/vdom/patch.js */
function invokeCreateHooks(vnode, insertedVnodeQueue) {
  for (let i = 0; i < cbs.create.length; ++i) {
    cbs.create[i](emptyNode, vnode)
  }
  i = vnode.data.hook // Reuse variable
  if (isDef(i)) {
    if (isDef(i.create)) i.create(emptyNode, vnode)
    if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
  }
}
```

可以看到，除了调用模块的`create`钩子之外，由于是组件`VNode`，所以存在`insert`钩子函数，所以最后还会将该组件添加到当前的`insertedVnodeQueue`中。

在调用完`initComponent`方法后，接着调用`insert`方法，将子组件的根节点插入到父节点中，这样就完成了组件`VNode`的渲染过程。

## 总结

`Vue`在执行`patch`的过程中，如果遇到组件`VNode`，会创建子组件的实例并进行挂载，这是一个深度递归的过程，当所有嵌套组件都完成挂载逻辑后，再将整个组件的根节点插入到父节点中，从而完成整个渲染逻辑。
