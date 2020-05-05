# [Vue源码学习] 响应式原理（中）

## 前言

从上一章节中，我们知道，在初始化`Vue`实例的过程中，会将`data`选项中的普通数据转换成响应式数据，但是如果不进行访问或设置，是无法起到效果的，那么接下来，就来看看`Vue`是如何进行依赖收集的。

## 观察者模式与依赖收集

在看具体代码之前，我们需要了解在`Vue`中是如何运用观察者模式和依赖收集的。

在观察者模式中，目标对象会管理一个集合，里面保存着所有依赖于该目标对象的观察者，当目标对象发生变化时，会通知集合中所有的观察者进行更新。在`Vue`中，目标对象就是每一个经过`defineReactive`方法处理过的数据(也包含`Observer`对象本身)，观察者就是一个个`Watcher`，它可以是渲染`Watcher`、计算`Watcher`、自定义`Watcher`等，而在调用`defineReactive`方法处理数据的过程中，会为每个数据创建一个`Dep`的实例，这个实例就是用来管理所有观察该目标对象的观察者集合，当数据发生变化时，`Vue`就会调用`dep.notify`方法，从而通知`Dep`中所有的观察者进行更新。

通过观察者模式，我们可以知道每个目标对象对应的观察者，而依赖收集解决的是每个观察者依赖了哪些数据，之所以这么设计，是为了在更新的过程中，通过重新计算观察者所依赖的目标数据，从而取消对多余的目标数据的观察，同时在目标数据的集合中移除该观察者，避免当目标数据再次发生变化时，产生额外的更新操作。那么接下来，就从源码的角度来看看`Vue`是如何实现该过程的。

从前面的**`$mount`**章节中，我们知道，每个组件实例都会创建一个渲染`Watcher`，在创建渲染`Watcher`的最后，会调用`get`方法，代码如下所示：

```js
/* core/observer/watcher.js */
export default class Watcher {
  constructor(
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    // ...
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  get() {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
}
```

```js
/* core/observer/dep.js */
Dep.target = null
const targetStack = []

export function pushTarget(target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget() {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

可以看到，在`get`方法中，首先会调用`pushTarget`方法，将当前`Watcher`赋值给`Dep.target`，然后调用`getter`方法，也就是在`mountComponent`方法中传入的`updateComponent`方法，在其中会调用组件的`render`渲染函数，而在创建`VNode`的过程中，如果需要访问`data`选项中的数据，那么就会触发数据的`get`访问器，代码如下所示：

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
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter(newVal) {
      // ...
    }
  })
}
```

可以看到，当触发`get`访问器时，此时的`Dep.target`指向的就是`pushTarget`的渲染`Watcher`，所以会调用此数据关联的`dep`的`depend`方法，进行依赖收集，代码如下所示：

```js
/* core/observer/dep.js */
export default class Dep {
  addSub(sub: Watcher) {
    this.subs.push(sub)
  }

  depend() {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
}
```

```js
/* core/observer/watcher.js */
export default class Watcher {
  addDep(dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
}
```

可以看到，`depend`方法并没有简单的调用`addSub`方法，将观察者添加到目标对象的集合中，而是继续调用`watcher.addDep`方法，在此方法中，首先将目标对象对应`dep`添加到`watcher.newDeps`中，这一步就是之前所说的依赖收集，如果检测到该`dep`不存在于上一次，该观察者所依赖的`watcher.depIds`中，说明这是一份新的依赖数据，才会调用`dep.addSub`方法，将该`watcher`实例添加到目标对象的`dep`集合中，完成观察者模式中的订阅操作。

所以在调用`dep.depend`方法的过程中，经过上面的步骤后，目标对象的`dep`中包含着`Watcher`实例，同时`Watcher`实例中同样也包含着目标对象的`dep`。在`get`访问器中，还有另一段逻辑，当`childOb`存在时，会继续调用`childOb.dep.depend`方法，从上一小节中可以知道，假如调用`defineReactive`方法，且对应的数据还是对象时，会继续调用`observe`方法，将嵌套对象转换为响应式对象，所以这里的`childOb`就是嵌套对象的`Observer`实例，这里的`childOb.dep`就是`Observer`实例对应的`dep`，在调用`depend`方法后，相当于将嵌套对象也添加到当前`Watcher`中，建立两者之间的联系，所以当嵌套对象直接修改时(`this.obj.nested = {}`)，也可以正确的触发更新操作。如果此时的`value`还是一个数组的话，会继续调用`dependArray`方法，在数组中的每项数据都与当前`Watcher`实例之间建立联系。

当`render`渲染函数执行完成后，当前组件的渲染`Watcher`就可以知道，它依赖了哪些数据，而这些数据中也保存有该渲染`Watcher`的引用。在调用完`getter`方法后，会调用`popTarget`方法，恢复`Dep.target`的引用，最后还会调用`cleanupDeps`方法，这个方法就是用来取消对多余的目标对象的观察，代码如下所示：

```js
/* core/observer/watcher.js */
export default class Watcher {
  cleanupDeps() {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }
}
```

在看`cleanupDeps`方法之前，需要了解`Watcher`对象中的`deps`、`newDeps`、`depIds`、`newDepIds`四个属性，在`Watcher`每次进行重新渲染之前，此时的`newDeps`、`newDepIds`是空的，而`deps`、`depIds`保存的是上一次渲染所依赖的数据，在渲染的过程中，`newDeps`和`newDepIds`会收集本次渲染所依赖的数据，同时将新增的数据添加到`deps`和`depIds`中，在渲染完成后，就会调用`cleanupDeps`方法，可以看到，此时`deps`、`depIds`中的数据可能会比`newDeps`、`newDepIds`中的数据多，这部分多出来的数据，其实就是该`Watcher`不需要依赖的数据，需要进行移除。

在`cleanupDeps`方法中，首先遍历`deps`对象中的所有数据，如果检测在`newDepIds`中不存在，那么就调用`dep.removeSub(this)`，从目标数据中的`dep`中移除当前`Watcher`，这样一来，当该数据再次发生变化时，就不会通知该`Watcher`了，然后交换`deps`与`newDeps`、`depIds`与`newDepIds`对象，然后将`newDeps`、`newDepIds`置空，这样就满足了在下次重新渲染时，`newDeps`、`newDepIds`是空的，`deps`、`depIds`保存的是上一次渲染所依赖的数据。

## 总结

`Vue`通过观察者模式和依赖收集，实现了在每次重新渲染的过程中，重新收集本次渲染所依赖的数据，这样一来，当依赖的数据发生变化时，就可以准确的通知相关的`Watcher`进行更新操作。
