# [Vue源码学习] _update（中）

## 前言

从上一章节中，我们知道，对于首次挂载，主要就是调用`createElm`方法，根据`VNode`渲染成真实`DOM`，那么接下来，就来看看该方法内部是如何实现该过程的。

## createElm

`createElm`方法的代码如下所示：

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
  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // This vnode was used in a previous render!
    // now it's used as a new node, overwriting its elm would cause
    // potential patch errors down the road when it's used as an insertion
    // reference node. Instead, we clone the node on-demand before creating
    // associated DOM element for it.
    vnode = ownerArray[index] = cloneVNode(vnode)
  }

  vnode.isRootInsert = !nested // for transition enter check
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }

  const data = vnode.data
  const children = vnode.children
  const tag = vnode.tag
  if (isDef(tag)) {
    if (process.env.NODE_ENV !== 'production') {
      if (data && data.pre) {
        creatingElmInVPre++
      }
      if (isUnknownElement(vnode, creatingElmInVPre)) {
        warn(
          'Unknown custom element: <' + tag + '> - did you ' +
          'register the component correctly? For recursive components, ' +
          'make sure to provide the "name" option.',
          vnode.context
        )
      }
    }

    vnode.elm = vnode.ns
      ? nodeOps.createElementNS(vnode.ns, tag)
      : nodeOps.createElement(tag, vnode)
    setScope(vnode)

    /* istanbul ignore if */
    if (__WEEX__) {
      // in Weex, the default insertion order is parent-first.
      // List items can be optimized to use children-first insertion
      // with append="tree".
      const appendAsTree = isDef(data) && isTrue(data.appendAsTree)
      if (!appendAsTree) {
        if (isDef(data)) {
          invokeCreateHooks(vnode, insertedVnodeQueue)
        }
        insert(parentElm, vnode.elm, refElm)
      }
      createChildren(vnode, children, insertedVnodeQueue)
      if (appendAsTree) {
        if (isDef(data)) {
          invokeCreateHooks(vnode, insertedVnodeQueue)
        }
        insert(parentElm, vnode.elm, refElm)
      }
    } else {
      createChildren(vnode, children, insertedVnodeQueue)
      if (isDef(data)) {
        invokeCreateHooks(vnode, insertedVnodeQueue)
      }
      insert(parentElm, vnode.elm, refElm)
    }

    if (process.env.NODE_ENV !== 'production' && data && data.pre) {
      creatingElmInVPre--
    }
  } else if (isTrue(vnode.isComment)) {
    vnode.elm = nodeOps.createComment(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  } else {
    vnode.elm = nodeOps.createTextNode(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  }
}
```

因为不同类型节点之间的差异很大，所以在渲染的时候，`Vue`将它们分成了四种类型，分别执行各自的逻辑：

1. 通过`createComponent`方法判断是否是组件`vnode`；

2. 通过`vnode.tag`判断是否是元素节点；

3. 通过`vnode.isComment`判断是否是注释节点；

4. 如果不是上述三种类型节点，一律当成文本节点；

其中组件`vnode`会在下一节中详细介绍，而注释节点和文本节点的逻辑很简单，就是调用`createComment`和`createTextNode`方法，创建真实`DOM`，然后调用`insert`方法，将该节点插入到父节点中，代码如下所示：

```js
/* platforms/web/runtime/node-ops.js */
export function createTextNode(text: string): Text {
  return document.createTextNode(text)
}

export function createComment(text: string): Comment {
  return document.createComment(text)
}

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

可以看到，这里主要是调用平台的原生方法，根据`vnode`生成真实`DOM`，那么接下来，就来看看`Vue`是如何生成元素节点的。

对于元素节点来说，`vnode.tag`就代表着标签名，所以首先通过调用`createElement`方法，创建对应的元素节点，并将其赋值给`vnode.elm`，代码如下所示：

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

export function createElementNS(namespace: string, tagName: string): Element {
  return document.createElementNS(namespaceMap[namespace], tagName)
}
```

`DOM`创建完成后，接着调用`createChildren`方法，根据`VNode.children`，递归地创建其子节点，代码如下所示：

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

可以看到，`createChildren`方法就是遍历`vnode.children`，继续使用`createElm`方法创建`VNode`对应的真实`DOM`，经过这个处理后，就可以根据组件的根`vnode`，生成整棵`DOM`树。

回到上面的`createElm`方法，在创建完子节点后，如果检测到包含`vnode.data`，就调用`invokeCreateHooks`方法，将`data`中的数据，绑定到元素上，代码如下所示：

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

其中，`cbs`包含了当前平台上，各个模块在不同执行阶段所对应的钩子函数，比如对`class`、`style`的处理方式不同，所以`Vue`将其拆分成独立的模块，同时暴露相同的接口，这样在外部就可以统一的进行管理，各个模块的具体内容在之后的章节中会详细介绍，然后检测当前`vnode`是否包含`create`、`insert`钩子函数，对其进行相应的处理。

经过上面的三个阶段，创建`DOM`节点，递归创建子节点，给节点绑定数据，此时已经完成了从`VNode`渲染成真实`DOM`的过程，最后，调用`insert`方法，将生成的节点插入到父节点中，整个元素节点的创建过程就完成了。

## 总结

`createElm`方法会根据不同的`vnode`类型，使用不同的方法创建真实的节点，对于元素节点来说，它会递归创建子节点，所以在给定一个`VNode`的情况下，`Vue`也能正确的生成整棵`DOM`树。
