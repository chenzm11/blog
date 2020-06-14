# [Vue源码学习] _update（中）

## 前言

在上一章节中，我们知道对于新的`vnode`，需要调用`createElm`方法渲染成真实的`DOM`，那么接下来，我们就看看其内部是如何实现的。

## createElm

`createElm`的代码如下所示：

```js
/* core/vdom/patch.js */
function createElm(
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  // ...

  vnode.isRootInsert = !nested // for transition enter check
  // 1. 组件VNode
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }

  const data = vnode.data
  const children = vnode.children
  const tag = vnode.tag
  if (isDef(tag)) {
    // 2. 普通元素节点
    vnode.elm = vnode.ns
      ? nodeOps.createElementNS(vnode.ns, tag)
      : nodeOps.createElement(tag, vnode)
    setScope(vnode)

    createChildren(vnode, children, insertedVnodeQueue)
    if (isDef(data)) {
      invokeCreateHooks(vnode, insertedVnodeQueue)
    }
    insert(parentElm, vnode.elm, refElm)

    if (process.env.NODE_ENV !== 'production' && data && data.pre) {
      creatingElmInVPre--
    }
  } else if (isTrue(vnode.isComment)) {
    // 3. 注释
    vnode.elm = nodeOps.createComment(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  } else {
    // 4. 文本
    vnode.elm = nodeOps.createTextNode(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  }
}
```

因为不同类型节点之间的差异很大，所以在渲染的时候，`Vue`将它们分成了四种类型，分别执行各自的逻辑：

1. 通过`createComponent`方法尝试从组件占位符`VNode`创建组件实例；

2. 通过`vnode.tag`判断是否是普通元素节点；

3. 通过`vnode.isComment`判断是否是注释节点；

4. 如果不是上述三种类型节点，一律当成文本节点；

我们首先来看看注释节点和文本节点的创建过程，`Vue`会调用平台相关的`createComment`和`createTextNode`方法，生成真实的`DOM`：

```js
/* platforms/web/runtime/node-ops.js */
export function createTextNode(text: string): Text {
  return document.createTextNode(text)
}

export function createComment(text: string): Comment {
  return document.createComment(text)
}
```

然后调用`insert`方法，将该节点插入到父节点中：

```js
/* core/vdom/patch.js */
function insert(parent, elm, ref) {
  if (isDef(parent)) {
    if (isDef(ref)) {
      if (nodeOps.parentNode(ref) === parent) {
        nodeOps.insertBefore(parent, elm, ref)
      }
    } else {
      nodeOps.appendChild(parent, elm)
    }
  }
}
```

由于这两种类型的节点没有`data`和`children`，所以到此也就处理完成了。那么接下来，我们就来看看`Vue`是如何生成元素节点的：

```js
// 1. 创建DOM
vnode.elm = vnode.ns
  ? nodeOps.createElementNS(vnode.ns, tag)
  : nodeOps.createElement(tag, vnode)
setScope(vnode)

// 2. 深度递归，创建子节点
createChildren(vnode, children, insertedVnodeQueue)

// 3. 执行module对应的create hook
if (isDef(data)) {
  invokeCreateHooks(vnode, insertedVnodeQueue)
}

// 4. 将节点插入到父节点中
insert(parentElm, vnode.elm, refElm)
```

可以看到，创建元素节点的逻辑还是很清晰的，首先调用`createElement`方法，创建对应的元素节点：

```js
/* platforms/web/runtime/node-ops.js */
export function createElement(tagName: string, vnode: VNode): Element {
  const elm = document.createElement(tagName)
  if (tagName !== 'select') {
    return elm
  }
  // false or null will remove the attribute but undefined will not
  if (vnode.data && vnode.data.attrs && vnode.data.attrs.multiple !== undefined) {
    elm.setAttribute('multiple', 'multiple')
  }
  return elm
}
```

然后调用`createChildren`方法，深度递归的创建子节点：

```js
/* core/vdom/patch.js */
function createChildren(vnode, children, insertedVnodeQueue) {
  if (Array.isArray(children)) {
    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(children)
    }
    for (let i = 0; i < children.length; ++i) {
      createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
    }
  } else if (isPrimitive(vnode.text)) {
    nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
  }
}
```

在所有的子节点都创建完成后，接着会判断`vnode.data`是否存在，如果存在，就会调用`invokeCreateHooks`方法，用来处理`modules`对应的`create`钩子函数：

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

可以看到，在`invokeCreateHooks`方法中，如果检测到`VNode`中存在`insert`钩子函数，还会将该`VNode`节点插入`insertedVnodeQueue`中，这是因为此时还在`VNode`的`create`阶段，所以需要等到`patch`结束后，在`invokeInsertHook`中再统一进行处理。

最后，将生成的`DOM`节点插入到父节点中，完成普通元素`VNode`的渲染过程。

## createComponent

除了上面三种情况外，其实一开始就会尝试通过`createComponent`方法，创建组件节点的实例，其代码如下所示：

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

可以看到，对于组件`VNode`，首先会从`data.hook`中取出`init`钩子函数，它是在创建组件`VNode`节点时，添加到`hook`上的，其代码如下所示：

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

在`init`方法中，首先会调用`createComponentInstanceForVnode`方法，创建子组件实例：

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

可以看到，在`createComponentInstanceForVnode`方法中，`Vue`首先在内部构建了一个配置对象，然后调用子组件的构造器，从而完成子组件的初始化过程，其中，配置对象中的`_parentVnode`表示子组件的父占位符`VNode`，它的`componentOptions`属性中包含了需要传给子实例的详细信息，`parent`表示子组件的父实例，也就是之后子实例的`$parent`，最后，通过`vnode.componentOptions.Ctor`子构造器，创建子组件的实例，在其内部，同样是调用`_init`方法，所以子组件的创建过程和根组件的创建过程是类似的。

创建好组件实例后，会就将其赋值给父占位符的`componentInstance`，然后调用`$mount`方法，挂载子组件实例，这部分逻辑也与之前类似，首先创建子组件的渲染`Watcher`，然后调用`_render`方法生成子组件的渲染`VNode`，接着调用`_update`方法根据`VNode`渲染成真实的`DOM`。

在调用完`init`钩子函数后，此时调用栈已经回到了父实例中，此时的`vnode.componentInstance`就指向了刚刚创建的子组件实例，所以接着调用`initComponent`方法：

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

可以看到，在`initComponent`方法中，首先尝试从`vnode`中提取`pendingInsert`，它里面包含了需要执行`insert`钩子函数的节点，然后将其添加到当前的`insertedVnodeQueue`中，初次渲染时，通过这样层层的提取，所以最后在根节点`patch`时，就可以在调用这些节点的`insert`钩子函数；接着，将子组件的根`DOM`赋值给父占位符的`elm`；然后判断子该组件是否是可`patchable`的，如果可挂载，那么就需要调用`invokeCreateHooks`方法，一方面将父占位符上的`data`作用在`DOM`中，另一方面，也将当前组件`vnode`插入到`insertedVnodeQueue`中。

最终，通过`createComponent`方法，同样也渲染成真实的`DOM`。

## 总结

`createElm`会根据`vnode`的类型，使用不同的方法创建真实的节点，对于元素节点来说，它会递归创建子节点，所以在给定一个根`vnode`的情况下，`Vue`也能正确的生成整棵`DOM`树，而对于组节点来说，它会通过`createComponent`方法，执行`_init`钩子函数，从而创建子组件的实例，并进行挂载，最终，同样可以渲染成真实的`DOM`。
