# [Vue源码学习] _update（上）

## 前言

在上一章节中，通过调用`_render`方法，最终返回了一个根`VNode`节点，那么接下来，就是调用`_update`方法，根据这个根`VNode`节点渲染成最终的`DOM`。

## _update

`_update`方法的代码如下所示：

```js
/* core/instance/lifecycle.js */
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const restoreActiveInstance = setActiveInstance(vm)
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  restoreActiveInstance()
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```

可以看到，首先从当前实例上获取`$el`和`_vnode`，它们分别表示上次的`DOM`元素和上次的`VNode`节点，对于根组件的首次渲染，此时的`vm.$el`就是调用`$mount`传入的第一个参数，而`vm._vnode`此时还未赋值，所以为`undefined`；然后调用`setActiveInstance`方法，设置当前正在渲染的`Vue`实例，其代码如下所示：

```js
/* core/instance/lifecycle.js */
export let activeInstance: any = null

export function setActiveInstance(vm: Component) {
  const prevActiveInstance = activeInstance
  activeInstance = vm
  return () => {
    activeInstance = prevActiveInstance
  }
}
```

可以看到，这里与上一章`_render`方法中的`currentRenderingInstance`不同，这里通过闭包维护了父子实例之间的关系，这是因为`_update`的过程不是线性的，而是一个深度递归的过程，在父实例执行`_update`过程中，会递归地执行子实例的`_update`，所以需要在子组件渲染完成后，重置`activeInstance`为父实例；然后将当前的`VNode`赋值给`vm._vnode`，这样在下次重新渲染时，就可以取到上一次的`VNode`；接着就是`_update`方法中的核心逻辑，调用`patch`方法进行渲染，通过判断是否是首次渲染，使用不同的参数调用`__patch__`方法，完成`VNode`到真实`DOM`的渲染，之后会详细介绍；完成渲染逻辑后，会将`patch`生成的真实`DOM`赋值给`vm.$el`，然后恢复对父实例的引用等逻辑。那么接下来，就详细看看`__patch__`方法是如何进行渲染的。

## patch

在看`patch`方法的实现之前，需要了解`patch`方法是如何构建出来的，因为对于`VNode`来说，它只是一个节点的描述，而不同的平台需要使用不同的方法进行渲染，所以对于`Web`平台来说，`patch`方法是在引入`Vue`时添加到原型上的，其代码如下所示：

```js
/* platforms/web/runtime/patch.js */
export const patch: Function = createPatchFunction({ nodeOps, modules })

/* platforms/web/runtime/index.js */
Vue.prototype.__patch__ = inBrowser ? patch : noop
```

可以看到，`patch`方法又是通过调用`createPatchFunction`方法构建的，而这里的`nodeOps`和`modules`变量，都是与当前平台相关的代码，所以不同的平台只需要使用与之对应的`nodeOps`和`modules`，就可以使用同一个`createPatchFunction`方法构建出当前平台的`patch`方法。`createPatchFunction`方法的源码在`core/vdom/patch.js`中，最终返回的`patch`方法如下所示：

```js
/* core/vdom/patch.js */
function patch(oldVnode, vnode, hydrating, removeOnly) {
  if (isUndef(vnode)) {
    if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
    return
  }

  let isInitialPatch = false
  const insertedVnodeQueue = []

  if (isUndef(oldVnode)) {
    // empty mount (likely as component), create new root element
    isInitialPatch = true
    createElm(vnode, insertedVnodeQueue)
  } else {
    const isRealElement = isDef(oldVnode.nodeType)
    if (!isRealElement && sameVnode(oldVnode, vnode)) {
      // patch existing root node
      patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
    } else {
      if (isRealElement) {
        // mounting to a real element
        // check if this is server-rendered content and if we can perform
        // a successful hydration.
        if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
          oldVnode.removeAttribute(SSR_ATTR)
          hydrating = true
        }
        if (isTrue(hydrating)) {
          if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
            invokeInsertHook(vnode, insertedVnodeQueue, true)
            return oldVnode
          } else if (process.env.NODE_ENV !== 'production') {
            warn(
              'The client-side rendered virtual DOM tree is not matching ' +
              'server-rendered content. This is likely caused by incorrect ' +
              'HTML markup, for example nesting block-level elements inside ' +
              '<p>, or missing <tbody>. Bailing hydration and performing ' +
              'full client-side render.'
            )
          }
        }
        // either not server-rendered, or hydration failed.
        // create an empty node and replace it
        oldVnode = emptyNodeAt(oldVnode)
      }

      // replacing existing element
      const oldElm = oldVnode.elm
      const parentElm = nodeOps.parentNode(oldElm)

      // create new node
      createElm(
        vnode,
        insertedVnodeQueue,
        // extremely rare edge case: do not insert if old element is in a
        // leaving transition. Only happens when combining transition +
        // keep-alive + HOCs. (#4590)
        oldElm._leaveCb ? null : parentElm,
        nodeOps.nextSibling(oldElm)
      )

      // update parent placeholder node element, recursively
      if (isDef(vnode.parent)) {
        let ancestor = vnode.parent
        const patchable = isPatchable(vnode)
        while (ancestor) {
          for (let i = 0; i < cbs.destroy.length; ++i) {
            cbs.destroy[i](ancestor)
          }
          ancestor.elm = vnode.elm
          if (patchable) {
            for (let i = 0; i < cbs.create.length; ++i) {
              cbs.create[i](emptyNode, ancestor)
            }
            // #6513
            // invoke insert hooks that may have been merged by create hooks.
            // e.g. for directives that uses the "inserted" hook.
            const insert = ancestor.data.hook.insert
            if (insert.merged) {
              // start at index 1 to avoid re-invoking component mounted hook
              for (let i = 1; i < insert.fns.length; i++) {
                insert.fns[i]()
              }
            }
          } else {
            registerRef(ancestor)
          }
          ancestor = ancestor.parent
        }
      }

      // destroy old node
      if (isDef(parentElm)) {
        removeVnodes([oldVnode], 0, 0)
      } else if (isDef(oldVnode.tag)) {
        invokeDestroyHook(oldVnode)
      }
    }
  }

  invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
  return vnode.elm
}
```

可以看到，`patch`方法还是比较复杂的，因为里面包含了创建、更新、移除、服务端渲染等逻辑。在`patch`方法中，首先判断是否存在`vnode`节点，如果不存在，说明当前组件没有需要渲染的内容，同时当`oldVnode`存在时，就调用`invokeDestroyHook`方法，执行`oldVnode`的卸载逻辑，此时就已经完成了渲染的操作；如果存在`vnode`节点，就需要做创建或更新的操作了，后面的所有逻辑，其实都是为此服务的。首先定义`isInitialPatch`和`insertedVnodeQueue`变量，它们用来确保组件的`insert`钩子函数在正确的时间点执行；接下来判断是否存在`oldVnode`节点，来决定到底使用何种方式来生成真实的`DOM`，当不存在`oldVnode`节点，说明是首次执行`patch`操作，比如子组件的初次`patch`，就直接调用`createElm`方法，根据`vnode`节点创建真实的`DOM`；当存在`oldVnode`节点，首先判断`oldVnode`节点是否是一个真实`DOM`节点，如果不是真实节点，并且新旧`vnode`又是`sameVnode`，说明此新旧`vnode`是相同的节点，那么就执行`patchVnode`方法，进行对比更新，减少开销，该方法会在之后的章节中再详细介绍；如果不满足此条件，说明新旧节点不是相同节点，无法进行对比，所以需要根据`vnode`重新创建`DOM`，首先判断如果`oldVnode`是一个真实`DOM`，就先调用`emptyNodeAt`方法，将其包装成`VNode`节点，代码如下所示：

```js
/* core/vdom/patch.js */
function emptyNodeAt(elm) {
  return new VNode(nodeOps.tagName(elm).toLowerCase(), {}, [], undefined, elm)
}
```

包装完成后，调用`createElm`方法，根据`vnode`创建真实的`DOM`，然后就是一些清理和更新的逻辑：

```js
/* core/vdom/patch.js */
function patch(oldVnode, vnode, hydrating, removeOnly) {
  if (isUndef(oldVnode)) {
    // ...
  } else {
      // update parent placeholder node element, recursively
      if (isDef(vnode.parent)) {
        let ancestor = vnode.parent
        const patchable = isPatchable(vnode)
        while (ancestor) {
          for (let i = 0; i < cbs.destroy.length; ++i) {
            cbs.destroy[i](ancestor)
          }
          ancestor.elm = vnode.elm
          if (patchable) {
            for (let i = 0; i < cbs.create.length; ++i) {
              cbs.create[i](emptyNode, ancestor)
            }
            // #6513
            // invoke insert hooks that may have been merged by create hooks.
            // e.g. for directives that uses the "inserted" hook.
            const insert = ancestor.data.hook.insert
            if (insert.merged) {
              // start at index 1 to avoid re-invoking component mounted hook
              for (let i = 1; i < insert.fns.length; i++) {
                insert.fns[i]()
              }
            }
          } else {
            registerRef(ancestor)
          }
          ancestor = ancestor.parent
        }
      }

      // destroy old node
      if (isDef(parentElm)) {
        removeVnodes([oldVnode], 0, 0)
      } else if (isDef(oldVnode.tag)) {
        invokeDestroyHook(oldVnode)
      }
    }
  }
}
```

可以看到，如果存在父占位符节点，首先需要在父占位符节点上调用所有模块的`destroy`钩子函数，卸载与旧`DOM`之间的联系，同时，如果新`DOM`是可`patchable`的，还需要在父占位符节点上调用所有模块的`create`钩子函数，构建与新`DOM`之间的联系。最后调用`removeVnodes`方法移除旧`DOM`或调用`invokeDestroyHook`方法卸载整个`oldVnode`。

完成以上的步骤后，此时已经根据`vnode`成功创建了`DOM`，最后调用`invokeInsertHook`方法，执行组件的`insert`钩子函数：

```js
/* core/vdom/patch.js */
function invokeInsertHook(vnode, queue, initial) {
  // delay insert hooks for component root nodes, invoke them after the
  // element is really inserted
  if (isTrue(initial) && isDef(vnode.parent)) {
    vnode.parent.data.pendingInsert = queue
  } else {
    for (let i = 0; i < queue.length; ++i) {
      queue[i].data.hook.insert(queue[i])
    }
  }
}

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

可以看到，对于初次渲染的子组件来说，并不会直接调用该组件对应的`insert`钩子函数，而是将其添加到其父占位符节点的`data.pendingInsert`中，在其父实例的`initComponent`方法中，又会将刚刚的`data.pendingInsert`添加到当前的`insertedVnodeQueue`中。这是因为在`updateComponent`过程中，在执行父组件的`patch`时，会穿插着执行子组件的初始化和挂载，而此时子组件的某些钩子函数，需要等到根组件渲染后才能调用，所以子组件通过`data.pendingInsert`属性将自身添加到父组件中，最终在渲染根组件时，就可以在同一帧中完成所有组件的`insert`钩子函数。

## 总结

在`_update`方法中，通过调用`patch`方法，可以根据`VNode`渲染成最终的`DOM`，并且整个过程是一个深度递归的过程，会在渲染`VNode`的过程中，执行子组件的初始化、挂载等逻辑。
