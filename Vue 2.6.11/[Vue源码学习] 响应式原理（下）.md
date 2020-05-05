# [Vue源码学习] 响应式原理（下）

## 前言

从之前的章节中，我们知道，`Vue`在执行渲染的过程中，会进行依赖收集的操作，那么在这一节中，就来看看当数据发生变化时，`Vue`是如何派发更新的。

## 派发更新

当数据发生变化时，会触发`set`访问器，代码如下所示：

```js
/* core/observer/index.js */
export function defineReactive(
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()
  // ...
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      // ...
    },
    set: function reactiveSetter(newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```

可以看到，在`set`访问器中，只有当新旧数据不相同时，才会执行之后的更新逻辑，如果传入的数据还是一个对象，同样也会调用`observe`方法，将数据转换为响应式对象，最后调用`dep.notify`方法，进行派发更新的操作，代码如下所示：

```js
/* core/observer/dep.js */
export default class Dep {
  notify() {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```

还记得在上一小节中，每个目标数据的`dep.subs`中保存着所有依赖于该目标对象的观察者，所以在`notify`方法中，遍历这些观察者，同时调用它们的`update`方法，进行更新操作，代码如下所示：

```js
/* core/observer/watcher.js */
export default class Watcher {
  update() {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
}
```

对于渲染`Watcher`来说，此时的`lazy`和`sync`都为`false`，所以会继续调用`queueWatcher`方法，代码如下所示：

```js
/* core/observer/scheduler.js */
export function queueWatcher(watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

可以看到，这里的`has`是用来确保在同一帧中，同一个`Watcher`实例只添加一次，而`queue`就是用来保存这些`Watcher`实例的地方，标志位`flushing`表示是否已经开始执行`Watcher`的更新操作，当`flushing`为`false`时，直接将`Watcher`实例添加到`queue`中，当`flushing`为`true`时，需要将`Watcher`实例添加到`queue`中特定的位置，标志位`waiting`表示在新的一轮更新中，将所有的同步更新操作，通过调用`nextTick`方法，延迟到下一帧中执行，从而避免更新数据时立即做`Watcher`的更新操作，提高性能。

当同步任务执行完后，此时的`queue`中保存着所有需要更新的`Watcher`实例，在下一帧中会调用`flushSchedulerQueue`方法，代码如下所示：

```js
/* core/observer/scheduler.js */
function flushSchedulerQueue() {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```

可以看到，在`flushSchedulerQueue`方法中，首先将标志位`flushing`置为`true`，表示已经开始执行`Watcher`的更新，然后对`queue`中的`Watcher`实例进行排序，按照创建`Watcher`的先后顺序，先创建的在前，后创建的在后，然后开始遍历`queue`中的`Watcher`列表，如果`Watcher`的选项中存在`before`钩子函数，那么在此时会先执行该函数，例如在渲染`Watcher`中，会执行`Vue`实例的`beforeUpdate`钩子函数，代码如下所示：

```js
/* core/instance/lifecycle.js */
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  // ...
  new Watcher(vm, updateComponent, noop, {
    before() {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  // ...
}
```

然后在执行`watcher.run`方法之前，将此`Watcher`实例从`has`中去除，然后就开始执行`Watcher`的更新操作，`run`方法的代码如下所示：

```js
/* core/observer/watcher.js */
export default class Watcher {
  run() {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }
}
```

可以看到，在`run`方法中，`Watcher`的更新其实还是调用`get`方法，对于渲染`Watcher`来说，就是重新执行`render`和`patch`操作，由于渲染`Watcher`调用`get`方法返回的`value`总是`undefined`，所以接下来的逻辑不会执行。

回到`flushSchedulerQueue`方法中，在执行完`watcher.run`方法后，在开发模式下会检测是否存在循环更新的操作，当在同一帧中同一个`Watcher`的更新次数大于`MAX_UPDATE_COUNT`时，会提示警告。当所有`Watcher`更新完毕后，`Vue`会做一些清理工作，将一些数据恢复成初始状态，首先会调用`resetSchedulerState`方法，代码如下所示：

```js
/* core/observer/scheduler.js */
function resetSchedulerState() {
  index = queue.length = activatedChildren.length = 0
  has = {}
  if (process.env.NODE_ENV !== 'production') {
    circular = {}
  }
  waiting = flushing = false
}
```

可以看到，在`resetSchedulerState`方法中，`Vue`将更新中使用到的数据恢复成初始状态，然后会调用`callUpdatedHooks`方法，代码如下所示：

```js
/* core/observer/scheduler.js */
function callUpdatedHooks(queue) {
  let i = queue.length
  while (i--) {
    const watcher = queue[i]
    const vm = watcher.vm
    if (vm._watcher === watcher && vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'updated')
    }
  }
}
```

在`callUpdatedHooks`方法中，`Vue`会在`queue`中找到所有渲染`Watcher`，然后通过这些渲染`Watcher`找到其对应的`Vue`实例，然后调用`updated`钩子函数。

此时，在同一轮事件循环中，所有的`Watcher`都已经更新完毕，而页面也已经得到了更新。

## 总结

`Vue`在修改数据的时候，首先会收集所有待更新的`Watcher`，然后在下一帧中，使用`flushSchedulerQueue`方法，同步更新这些`Watcher`。
