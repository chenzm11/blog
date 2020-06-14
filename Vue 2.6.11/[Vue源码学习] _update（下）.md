# [Vue源码学习] _update（下）

在上一章节中，我们可以通过`createElm`将`VNode`渲染成真实的`DOM`，那么在本章节中，我们就来看看对于相同节点，`Vue`是如何进行对比更新的。

## patchVnode

在组件更新的过程中，首先还是会调用`_render`方法，根据当前帧的状态，生成组件的渲染`vnode`，然后调用`_update`方法，由于这次是更新操作，所以可以取到新旧`vnode`节点，然后调用`patch`方法，在该方法中就会调用`sameVnode`判断新旧`vnode`是否是相同节点：

```js
/* core/vdom/patch.js */
function sameVnode(a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

如果是相同节点，则说明渲染根节点是可以复用的，不需要调用`createElm`渲染一个全新的`DOM`，所以此时就会调用`patchVnode`方法，进行节点的对比更新操作：

```js
/* core/vdom/patch.js */
function patchVnode(
  oldVnode,
  vnode,
  insertedVnodeQueue,
  ownerArray,
  index,
  removeOnly
) {
  if (oldVnode === vnode) {
    return
  }

  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // clone reused vnode
    vnode = ownerArray[index] = cloneVNode(vnode)
  }

  // 重用DOM
  const elm = vnode.elm = oldVnode.elm

  if (isTrue(oldVnode.isAsyncPlaceholder)) {
    if (isDef(vnode.asyncFactory.resolved)) {
      hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
    } else {
      vnode.isAsyncPlaceholder = true
    }
    return
  }

  // reuse element for static trees.
  // note we only do this if the vnode is cloned -
  // if the new node is not cloned it means the render functions have been
  // reset by the hot-reload-api and we need to do a proper re-render.
  if (isTrue(vnode.isStatic) &&
    isTrue(oldVnode.isStatic) &&
    vnode.key === oldVnode.key &&
    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
  ) {
    vnode.componentInstance = oldVnode.componentInstance
    return
  }

  // 组件的占位符节点在这里会调用prepatch钩子函数
  let i
  const data = vnode.data
  if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
    i(oldVnode, vnode)
  }

  // 针对data中的数据，调用各modules对应的update钩子函数
  const oldCh = oldVnode.children
  const ch = vnode.children
  if (isDef(data) && isPatchable(vnode)) {
    for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
    if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
  }

  // 根据子节点的状态，使用不同的方式更新子节点
  if (isUndef(vnode.text)) {
    if (isDef(oldCh) && isDef(ch)) {
      if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
    } else if (isDef(ch)) {
      if (process.env.NODE_ENV !== 'production') {
        checkDuplicateKeys(ch)
      }
      if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
      addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
    } else if (isDef(oldCh)) {
      removeVnodes(oldCh, 0, oldCh.length - 1)
    } else if (isDef(oldVnode.text)) {
      nodeOps.setTextContent(elm, '')
    }
  } else if (oldVnode.text !== vnode.text) {
    nodeOps.setTextContent(elm, vnode.text)
  }
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
  }
}
```

`patchVnode`方法虽然看上去很复杂，但是只要明白了`patchVnode`的含义，就很容易理解`Vue`为什么这么处理了。

首先只要调用`patchVnode`方法，则说明当前新旧`vnode`是可以复用的，既然可以复用，那么可以直接将`oldVnode.elm`赋值给`vnode.elm`，这样就省去了新建`DOM`的消耗，然后在前面的章节中，我们知道对于一个`VNode`来说，它最重要的是`tag`、`data`、`children`，现在`tag`对应的`DOM`已经复用了，那么接下来就只需要处理`data`和`children`就可以了。

所以接下来就根据`vnode.data`中的数据，调用各`modules`对应的`update`钩子函数，将最新的数据更新到`DOM`上，这样就完成了`data`的更新操作，接着就根据新旧节点的`children`，使用不同的方式更新子节点：

```js
if (isUndef(vnode.text)) {
  if (isDef(oldCh) && isDef(ch)) {
    // 新旧节点的children都存在时，调用updateChildren对比子节点
    if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
  } else if (isDef(ch)) {
    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(ch)
    }
    // 新节点存在children，旧节点不存在children，删除文本后调用addVnodes创建全新的子节点
    if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
    addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
  } else if (isDef(oldCh)) {
    // 新节点不存在children，也不存在text，旧节点存在children，调用removeVnodes删除所有的子节点
    removeVnodes(oldCh, 0, oldCh.length - 1)
  } else if (isDef(oldVnode.text)) {
    // 新节点不存在children，也不存在text，旧节点存在text，调用setTextContent删除节点文本
    nodeOps.setTextContent(elm, '')
  }
} else if (oldVnode.text !== vnode.text) {
  // vnode是文本节点，并且与oldVnode.text不相同时，调用setTextContent删除节点文本
  nodeOps.setTextContent(elm, vnode.text)
}
```

从上面可以看到，只有当新旧节点的`children`都存在时，才会调用`updateChildren`方法，对子节点进行对比更新，其余的三种操作都很简单：

```js
/* core/vdom/patch.js */
function addVnodes(parentElm, refElm, vnodes, startIdx, endIdx, insertedVnodeQueue) {
  for (; startIdx <= endIdx; ++startIdx) {
    createElm(vnodes[startIdx], insertedVnodeQueue, parentElm, refElm, false, vnodes, startIdx)
  }
}

function removeVnodes(vnodes, startIdx, endIdx) {
  for (; startIdx <= endIdx; ++startIdx) {
    const ch = vnodes[startIdx]
    if (isDef(ch)) {
      if (isDef(ch.tag)) {
        removeAndInvokeRemoveHook(ch)
        invokeDestroyHook(ch)
      } else { // Text node
        removeNode(ch.elm)
      }
    }
  }
}
```

```js
/* platforms/web/runtime/node-ops.js */
export function setTextContent(node: Node, text: string) {
  node.textContent = text
}
```

`addVnodes`方法根据`vnode.children`生成全新的`DOM`节点，`removeVnodes`方法根据`oldVnode.children`卸载所有的`vnode`，`setTextContent`方法直接设置节点的`textContent`。

那么接下来，我们就来详细看看`updateChildren`方法的具体实现。

## updateChildren

```js
/* core/vdom/patch.js */
function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  let oldStartIdx = 0
  let newStartIdx = 0
  let oldEndIdx = oldCh.length - 1
  let oldStartVnode = oldCh[0]
  let oldEndVnode = oldCh[oldEndIdx]
  let newEndIdx = newCh.length - 1
  let newStartVnode = newCh[0]
  let newEndVnode = newCh[newEndIdx]
  let oldKeyToIdx, idxInOld, vnodeToMove, refElm

  // removeOnly is a special flag used only by <transition-group>
  // to ensure removed elements stay in correct relative positions
  // during leaving transitions
  const canMove = !removeOnly

  if (process.env.NODE_ENV !== 'production') {
    checkDuplicateKeys(newCh)
  }

  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (isUndef(oldStartVnode)) {
      oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
    } else if (isUndef(oldEndVnode)) {
      oldEndVnode = oldCh[--oldEndIdx]
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      oldStartVnode = oldCh[++oldStartIdx]
      newStartVnode = newCh[++newStartIdx]
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      oldEndVnode = oldCh[--oldEndIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
      oldStartVnode = oldCh[++oldStartIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
      oldEndVnode = oldCh[--oldEndIdx]
      newStartVnode = newCh[++newStartIdx]
    } else {
      if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
      idxInOld = isDef(newStartVnode.key)
        ? oldKeyToIdx[newStartVnode.key]
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
      if (isUndef(idxInOld)) { // New element
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
      } else {
        vnodeToMove = oldCh[idxInOld]
        if (sameVnode(vnodeToMove, newStartVnode)) {
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
          oldCh[idxInOld] = undefined
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
        } else {
          // same key but different element. treat as new element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        }
      }
      newStartVnode = newCh[++newStartIdx]
    }
  }
  if (oldStartIdx > oldEndIdx) {
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
  } else if (newStartIdx > newEndIdx) {
    removeVnodes(oldCh, oldStartIdx, oldEndIdx)
  }
}
```

`updateChildren`内部使用了双端比较算法，首先定义`newStartIdx`、`newEndIdx`、`oldStartIdx`、`oldEndIdx`四个索引，然后保存它们对应的`VNode`，然后在运行的时候，尝试做如下的四种比较：

1. `oldStartVnode`和`newStartVnode`：

    如果检测是`sameVnode`，首先调用`patchVnode`方法，进行对比更新操作，然后将`newStartIdx`和`oldStartIdx`前进一位，同时更新`oldStartVnode`和`newStartVnode`。

2. `oldEndVnode`和`newEndVnode`：

    如果检测是`sameVnode`，首先调用`patchVnode`方法，进行对比更新操作，然后将`newEndIdx`和`oldEndIdx`后退一位，同时更新`oldEndVnode`和`newEndVnode`。

3. `oldStartVnode`和`newEndVnode`：

    如果检测是`sameVnode`，首先调用`patchVnode`方法，进行对比更新操作，由于此时`oldEndIdx`后面的节点已经是排好序的，所以只需要将`oldStartVnode`移动到`oldEndVnode`的后面即可，然后将`oldStartIdx`前进一位，将`newEndIdx`后退一位，同时更新`oldStartVnode`和`newEndVnode`。

4. `oldEndVnode`和`newStartVnode`：

    如果检测是`sameVnode`，首先调用`patchVnode`方法，进行对比更新操作，由于此时`oldStartIdx`前面的节点已经是排好序的，所以只需要将`oldEndVnode`移动到`oldStartVnode`的前面即可，然后将`oldEndIdx`后退一位，将`newStartIdx`前进一位，同时更新`oldEndVnode`和`newStartVnode`。

如果以上情况都不满足，那么对于`newStartVnode`来说，要么`sameVnode`处于`oldCh`的中间位置，要么是一个全新的节点，所以首先会调用`createKeyToOldIdx`方法，从剩余的`oldCh`中提取`key`与`index`对应的哈希表：

```js
function createKeyToOldIdx(children, beginIdx, endIdx) {
  let i, key
  const map = {}
  for (i = beginIdx; i <= endIdx; ++i) {
    key = children[i].key
    if (isDef(key)) map[key] = i
  }
  return map
}
```

然后尝试通过`newStartVnode.key`从哈希表中找到对应的节点，如果没有找到，说明`newStartVnode`是一个全新的节点，就调用`createElm`方法创建真实的节点，如果找到的话，首先还是通过`sameVnode`判断这两个节点是否是相同节点，如果是相同节点，就调用`patchVnode`方法，进行对比更新操作，然后将该节点移动到`oldStartVnode`的前面，同时将`oldCh[idxInOld]`置为`undefined`，表示该节点已经处理过了，在遍历的时候需要跳过该节点；如果不是相同节点，就调用`createElm`方法创建真实的节点。处理完成后，将`newStartIdx`前进一位，同时更新`newStartVnode`。

在双端比较的过程中，如果遇到`oldStartIdx`大于`oldEndIdx`或`newStartIdx`大于`newEndIdx`的时候，说明`oldCh`或`newCh`至少有一个已经遍历完，而没有遍历完的节点，在`oldCh`中表示多余的节点，需要删除，在`newCh`中表示新增的节点，需要添加，所以在方法的最后，还会执行这么一段逻辑，对剩余的节点做处理：

```js
if (oldStartIdx > oldEndIdx) {
  refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
  addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
} else if (newStartIdx > newEndIdx) {
  removeVnodes(oldCh, oldStartIdx, oldEndIdx)
}
```

可以看到，在`updateChildren`方法中，调用`patchVnode`和`createElm`，都是一个深度递归的过程，所以最终整棵`DOM`树都会得到更新。

在上面的`patchVnode`中，还有一段处理组件`VNode`的逻辑，`prepatch`钩子函数，接下来，我们就来看看组件是如何更新的。

## prepatch hook

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

在前面创建组件`VNode`的过程中，我们知道组件占位符节点会将`propsData`、`listeners`、`children`保存在`componentOptions`中，在创建子实例时，占位符节点会将这些数据传递给子实例，所以在调用`prepatch`钩子函数时，也只需要传入`propsData`、`listeners`、`children`，对原始的数据进行更新即可，`updateChildComponent`的定义如下所示：

```js
/* core/instance/lifecycle.js */
export function updateChildComponent(
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
  if (process.env.NODE_ENV !== 'production') {
    isUpdatingChildComponent = true
  }

  // determine whether component has slot children
  // we need to do this before overwriting $options._renderChildren.

  // check if there are dynamic scopedSlots (hand-written or compiled but with
  // dynamic slot names). Static scoped slots compiled from template has the
  // "$stable" marker.
  const newScopedSlots = parentVnode.data.scopedSlots
  const oldScopedSlots = vm.$scopedSlots
  const hasDynamicScopedSlot = !!(
    (newScopedSlots && !newScopedSlots.$stable) ||
    (oldScopedSlots !== emptyObject && !oldScopedSlots.$stable) ||
    (newScopedSlots && vm.$scopedSlots.$key !== newScopedSlots.$key)
  )

  // 只有普通插槽和动态插槽需要调用$forceUpdate方法，单纯的作用域插是不会在这里
  // 调用$forceUpdate的，因为作用域插槽是在子组件内部渲染的，如果插槽内部使用了
  // 父组件中的数据，那么根据响应式系统，该数据的dep中会将子组件的渲染Watcher加
  // 入到它的观察者列表中，所以当该数据发生改变时，就已经将子组件添加到更新列表
  // 了，所以不需要调用$forceUpdate，同时，如果该数据没有发生改变，那么子组件也
  // 不需要重新渲染作用域插槽中的内容，调用$forceUpdate也是浪费的。

  // Any static slot children from the parent may have changed during parent's
  // update. Dynamic scoped slots may also have changed. In such cases, a forced
  // update is necessary to ensure correctness.
  const needsForceUpdate = !!(
    renderChildren ||               // has new static slots
    vm.$options._renderChildren ||  // has old static slots
    hasDynamicScopedSlot
  )

  vm.$options._parentVnode = parentVnode
  vm.$vnode = parentVnode // update vm's placeholder node without re-render

  if (vm._vnode) { // update child tree's parent
    vm._vnode.parent = parentVnode
  }
  vm.$options._renderChildren = renderChildren

  // update $attrs and $listeners hash
  // these are also reactive so they may trigger child update if the child
  // used them during render
  vm.$attrs = parentVnode.data.attrs || emptyObject
  vm.$listeners = listeners || emptyObject

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

  // update listeners
  listeners = listeners || emptyObject
  const oldListeners = vm.$options._parentListeners
  vm.$options._parentListeners = listeners
  updateComponentListeners(vm, listeners, oldListeners)

  // resolve slots + force update if has children
  if (needsForceUpdate) {
    vm.$slots = resolveSlots(renderChildren, parentVnode.context)
    vm.$forceUpdate()
  }

  if (process.env.NODE_ENV !== 'production') {
    isUpdatingChildComponent = false
  }
}
```

在`updateChildComponent`方法中，里面的`$forceUpdate`是与插槽相关的，这在之后插槽的章节中再详细介绍，我们先来看看除了插槽以外的内容，可以看到，除了更新引用关系外，还有三段逻辑：

1. 更新`$attrs`、`$listeners`：

    ```js
    vm.$attrs = parentVnode.data.attrs || emptyObject
    vm.$listeners = listeners || emptyObject
    ```

    在前面的`initRender`方法中，在实例上定义了这两个响应式属性，所以如果在组件内部使用了这两个属性，那么此时就会通知组件需要做更新操作。

2. 更新`propsData`：

    ```js
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
    ```

    组件的`props`同样也是响应式属性，所以如果这些属性在父组件中更新的话，就会通知子组件需要做更新操作。

3. 更新`listeners`：

    ```js
    listeners = listeners || emptyObject
    const oldListeners = vm.$options._parentListeners
    vm.$options._parentListeners = listeners
    updateComponentListeners(vm, listeners, oldListeners)
    ```

    调用`updateComponentListeners`方法更新事件。

`updateChildComponent`方法除了更新组件实例上的属性外，最主要的作用就是在这些数据发生变化时，可以通知子组件需要做更新操作，将子组件的渲染`Watcher`添加到`queueWatcher`中，从而在更新完父组件后，子组件同样也会派发更新操作。
