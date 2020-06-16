# [Vue源码学习] 插槽（上）

## 前言

在`Vue`中，可以通过插槽实现内容的分发，在`2.6.0`版本之前，是通过`slot`和`slot-scope`选项，来区分普通插槽和作用域插槽，它们之间有很大的区别，普通插槽是在父实例执行`render`的过程中，立即创建插槽对应的`VNode`，而作用域插槽是在子实例执行`render`的过程中，通过`normalizeScopedSlots`方法解析`data.scopedSlots`创建其对应的`VNode`，为了统一实现插槽的功能，所以在`2.6.0`版本之后，`Vue`提供了`v-slot`指令，用来统一普通插槽和作用域插槽，现在只要使用`v-slot`指令，那么其内部的内容，就会自动定义为作用域插槽，也就是说，它们都是在子实例执行`render`的过程中，才去创建其对应的`VNode`，那么接下来，就根据一个简单的例子，来看看`v-slot`是如何实现插槽功能的，示例如下所示：

```js
Vue.component('parent', {
  template: `
<div class="parent">
  <child>
    <template v-slot:header>
      <div>Parent Heaader</div>
    </template>
    <template v-slot:default="slotProps">
      <div>Parent {{slotProps.message}}</div>
    </template>
    <template v-slot:footer>
      <div>Parent Footer</div>
    </template>
  </child>
</div>
    `
})
```

```js
Vue.component('child', {
  template: `
<div class="child">
  <slot name="header">
    <div>Header</div>
  </slot>
  <slot :message="message">
    <div>Main</div>
  </slot>
  <slot name="footer">
    <div>Footer</div>
  </slot>
</div>
`,
  data() {
    return {
      message: 'Main'
    }
  }
})
```

## v-slot

首先来看看在编译的过程中，`Vue`是如何处理父模板中定义的`v-slot`指令的。

在编译的`parse`阶段，在解析开始标签时，会将标签上的每对属性保存在`AST`的`attrsList`和`attrsMap`中，当解析到对应的结束标签时，会调用`processElement`方法来处理`AST`上的属性，代码如下所示：

```js
/* compiler/parser/index.js */
function closeElement(element) {
  // ...
  if (!inVPre && !element.processed) {
    element = processElement(element, options)
  }
  // ...
}

export function processElement(
  element: ASTElement,
  options: CompilerOptions
) {
  // ...
  processSlotContent(element)
  // ...
}
```

对于包含`v-slot`指令的`AST`节点来说，它会调用`processSlotContent`方法做进一步的处理，代码如下所示：

```js
/* compiler/parser/index.js */
export const emptySlotScopeToken = `_empty_`

const slotRE = /^v-slot(:|$)|^#/

function processSlotContent(el) {
  let slotScope
  if (el.tag === 'template') {
    slotScope = getAndRemoveAttr(el, 'scope')
    // ...
    el.slotScope = slotScope || getAndRemoveAttr(el, 'slot-scope')
  } else if ((slotScope = getAndRemoveAttr(el, 'slot-scope'))) {
    // ...
    el.slotScope = slotScope
  }

  // slot="xxx"
  const slotTarget = getBindingAttr(el, 'slot')
  if (slotTarget) {
    el.slotTarget = slotTarget === '""' ? '"default"' : slotTarget
    el.slotTargetDynamic = !!(el.attrsMap[':slot'] || el.attrsMap['v-bind:slot'])
    // preserve slot as an attribute for native shadow DOM compat
    // only for non-scoped slots.
    if (el.tag !== 'template' && !el.slotScope) {
      addAttr(el, 'slot', slotTarget, getRawBindingAttr(el, 'slot'))
    }
  }

  // 2.6 v-slot syntax
  if (process.env.NEW_SLOT_SYNTAX) {
    if (el.tag === 'template') {
      // v-slot on <template>
      const slotBinding = getAndRemoveAttrByRegex(el, slotRE)
      if (slotBinding) {
        // ...
        const { name, dynamic } = getSlotName(slotBinding)
        el.slotTarget = name
        el.slotTargetDynamic = dynamic
        el.slotScope = slotBinding.value || emptySlotScopeToken // force it into a scoped slot for perf
      }
    } else {
      // v-slot on component, denotes default slot
      const slotBinding = getAndRemoveAttrByRegex(el, slotRE)
      if (slotBinding) {
        // ...
        // add the component's children to its default slot
        const slots = el.scopedSlots || (el.scopedSlots = {})
        const { name, dynamic } = getSlotName(slotBinding)
        const slotContainer = slots[name] = createASTElement('template', [], el)
        slotContainer.slotTarget = name
        slotContainer.slotTargetDynamic = dynamic
        slotContainer.children = el.children.filter((c: any) => {
          if (!c.slotScope) {
            c.parent = slotContainer
            return true
          }
        })
        slotContainer.slotScope = slotBinding.value || emptySlotScopeToken
        // remove children as they are returned from scopedSlots now
        el.children = []
        // mark el non-plain so data gets generated
        el.plain = false
      }
    }
  }
}

function getSlotName(binding) {
  let name = binding.name.replace(slotRE, '')
  if (!name) {
    if (binding.name[0] !== '#') {
      name = 'default'
    } else if (process.env.NODE_ENV !== 'production') {
      warn(
        `v-slot shorthand syntax requires a slot name.`,
        binding
      )
    }
  }
  return dynamicArgRE.test(name)
    // dynamic [name]
    ? { name: name.slice(1, -1), dynamic: true }
    // static name
    : { name: `"${name}"`, dynamic: false }
}
```

可以看到，在`processSlotContent`方法中，前面的两段代码是用来兼容`2.6.0`之前的逻辑，用来处理了`scope`、`slot-scope`、`slot`，之后的逻辑就是对新指令`v-slot`的处理。因为`v-slot`指令不仅可以添加到第一级`<template>`标签上，还可以添加到组件本身，所以`Vue`使用`el.tag === 'template'`来判断具体是哪一种情况，来做不同的处理。

* 如果`v-slot`指令在`<template>`标签上，首先通过`getAndRemoveAttrByRegex`方法从`AST.attrsList`中获取`v-slot`对应的数据，如果存在`v-slot`指令，就通过`getSlotName`方法，从参数中提取对应的`slotTarget`和`slotTargetDynamic`，最后通过`slotBinding.value`取出对应的`slotScope`，如果`v-slot`指令中不包含插槽数据`slotProps`，就使用`emptySlotScopeToken`代替。例如`<template v-slot:default="slotProps"></template>`模板，取出的`slotTarget`就是`default`，`slotTargetDynamic`就是`false`，`slotScope`就是`slotProps`。

* 如果`v-slot`指令在组件上，这时会创建一个新的`AST`节点`slotContainer`，将原先组件中的插槽内容移到`slotContainer`中，这样就如同前一种情况，在`slotContainer`上添加`slotTarget`、`slotTargetDynamic`、`slotScope`，不同的是，在构造完成后，会将`slotContainer`添加到组件`AST`的`scopedSlots`属性中，同时将组件`AST`的`children`置为空。

通过上面的处理，可以发现，只要使用了`v-slot`指令，此时的`slotScope`总是会取得一个值，在没有手动设置值时也会取得默认值`emptySlotScopeToken`，正是通过这个处理，`Vue`才可以统一使用作用域插槽。

调用完`processElement`方法后，此时`AST`节点上已经添加了`slotTarget`、`slotTargetDynamic`、`slotScope`等属性，回到最开始的`closeElement`方法：

```js
/* compiler/parser/index.js */
function closeElement(element) {
  // ...
  if (currentParent && !element.forbidden) {
    if (element.elseif || element.else) {
      // ...
    } else {
      if (element.slotScope) {
        // scoped slot
        // keep it in the children list so that v-else(-if) conditions can
        // find it as the prev node.
        const name = element.slotTarget || '"default"'
          ; (currentParent.scopedSlots || (currentParent.scopedSlots = {}))[name] = element
      }
      currentParent.children.push(element)
      element.parent = currentParent
    }
  }

  // final children cleanup
  // filter out scoped slots
  element.children = element.children.filter(c => !(c: any).slotScope)
  // ...
}
```

可以看到，当`v-slot`指令在`<template>`标签上时，`template`对应的`AST`节点会具有`slotScope`属性，那么就会在父`AST`节点，也就是组件`AST`节点的`scopedSlots`对象上，将当前`template`节点作为插槽添加进去。当所有的子节点都完成闭合操作后，就会执行父节点的闭合操作，由于插槽在本质上不属于子节点，而且在处理子节点的过程中，已经将插槽添加到父节点的`scopedSlots`中，所以在父节点闭合时会执行`element.children.filter(c => !(c: any).slotScope)`，将所有的作用域插槽从父节点的`children`中移除，这样一来，作用域插槽就只能通过父节点的`scopedSlots`属性进行访问了。

在将模板转换成`AST`节点后，就会进行编译的`generate`阶段，用来将`AST`节点转换成最终的代码，在这个过程中，会调用`genData`方法，用来处理`AST`上的数据，当检测到`AST`上存在`scopedSlots`属性时，就会调用`genScopedSlots`方法来处理插槽，代码如下所示：

```js
/* compiler/codegen/index.js */
export function genData(el: ASTElement, state: CodegenState): string {
  // ...
  // scoped slots
  if (el.scopedSlots) {
    data += `${genScopedSlots(el, el.scopedSlots, state)},`
  }
  // ...
}

function genScopedSlots(
  el: ASTElement,
  slots: { [key: string]: ASTElement },
  state: CodegenState
): string {
  // ...
  const generatedSlots = Object.keys(slots)
    .map(key => genScopedSlot(slots[key], state))
    .join(',')

  return `scopedSlots:_u([${generatedSlots}]${
    needsForceUpdate ? `,null,true` : ``
    }${
    !needsForceUpdate && needsKey ? `,null,false,${hash(generatedSlots)}` : ``
    })`
}

function genScopedSlot(
  el: ASTElement,
  state: CodegenState
): string {
  const isLegacySyntax = el.attrsMap['slot-scope']
  // ...
  const slotScope = el.slotScope === emptySlotScopeToken
    ? ``
    : String(el.slotScope)
  const fn = `function(${slotScope}){` +
    `return ${el.tag === 'template'
      ? el.if && isLegacySyntax
        ? `(${el.if})?${genChildren(el, state) || 'undefined'}:undefined`
        : genChildren(el, state) || 'undefined'
      : genElement(el, state)
    }}`
  // reverse proxy v-slot without scope on this.$slots
  const reverseProxy = slotScope ? `` : `,proxy:true`
  return `{key:${el.slotTarget || `"default"`},fn:${fn}${reverseProxy}}`
}
```

可以看到，在`genScopedSlots`方法中，就是遍历`scopedSlots`，然后调用`genScopedSlot`方法，将生成`VNode`的代码封装在一个函数中，同时，将`slotScope`的值作为此函数的第一个形参，所以在`2.6.0`中，只要使用了`v-slot`指令，插槽就会成为生成一个函数，也就是作用域插槽的形式，遍历完`scopedSlots`后，将生成的插槽数组用`_u`方法包裹，该方法会在运行时中再详细介绍。

经过`parse`和`generate`阶段的处理，`Vue`已经成功完成对父模板的编译过程，示例代码中对应的渲染函数如下所示：

```js
function anonymous() {
  with (this) {
    return _c('div', {
      staticClass: "parent"
    }, [_c('child', {
      scopedSlots: _u([{
        key: "header",
        fn: function () {
          return [_c('div', [_v("Parent Heaader")])]
        },
        proxy: true
      }, {
        key: "default",
        fn: function (slotProps) {
          return [_c('div', [_v("Parent " + _s(slotProps.message))])]
        }
      }, {
        key: "footer",
        fn: function () {
          return [_c('div', [_v("Parent Footer")])]
        },
        proxy: true
      }])
    })], 1)
  }
}
```

上面的`proxy`属性表示虽然这里是作用域插槽的形式，但是在运行时需要将其代理到`$slot`中。

`v-slot`指令是在父组件中使用的，接下来就来看看在子组件中，`Vue`是如何处理`<slot>`标签的。

## slot标签

同样是在编译的`parse`阶段，在处理结束标签时，会执行`processElement`方法，然后在其中又会执行`processSlotOutlet`方法，用来处理`<slot>`标签，代码如下所示：

```js
/* compiler/parser/index.js */
export function processElement(
  element: ASTElement,
  options: CompilerOptions
) {
  // ...
  processSlotOutlet(element)
  // ...
}

function processSlotOutlet(el) {
  if (el.tag === 'slot') {
    el.slotName = getBindingAttr(el, 'name')
    if (process.env.NODE_ENV !== 'production' && el.key) {
      warn(
        `\`key\` does not work on <slot> because slots are abstract outlets ` +
        `and can possibly expand into multiple elements. ` +
        `Use the key on a wrapping element instead.`,
        getRawBindingAttr(el, 'key')
      )
    }
  }
}
```

可以看到，`processSlotOutlet`方法就是在`<slot>`标签上获取`name`选项，并赋值给`AST`的`slotName`属性。然后在编译的`generate`阶段，在`genElement`方法中，会对`slot`标签做特殊处理，代码如下所示：

```js
/* compiler/codegen/index.js */
export function genElement(el: ASTElement, state: CodegenState): string {
  // ...
  if (el.tag === 'slot') {
    return genSlot(el, state)
  }
  // ...
}

function genSlot(el: ASTElement, state: CodegenState): string {
  const slotName = el.slotName || '"default"'
  const children = genChildren(el, state)
  let res = `_t(${slotName}${children ? `,${children}` : ''}`
  const attrs = el.attrs || el.dynamicAttrs
    ? genProps((el.attrs || []).concat(el.dynamicAttrs || []).map(attr => ({
      // slot props are camelized
      name: camelize(attr.name),
      value: attr.value,
      dynamic: attr.dynamic
    })))
    : null
  const bind = el.attrsMap['v-bind']
  if ((attrs || bind) && !children) {
    res += `,null`
  }
  if (attrs) {
    res += `,${attrs}`
  }
  if (bind) {
    res += `${attrs ? '' : ',null'},${bind}`
  }
  return res + ')'
}
```

可以看到，当遇到`slot`标签时，会调用`genSlot`方法，在该方法中，首先会使用`slotName`作为插槽的名称，然后调用`genChildren`方法，生成此插槽对应的后备内容，然后调用`genProps`方法，从`AST`上获取待传入插槽函数的数据作为实参，最后使用`_t`方法包裹，该方法同样是在运行时中用来处理`VNode`节点的，之后再进行详细的分析。

经过上面的处理，`Vue`已经成功完成对子模板的编译过程，示例代码中对应的渲染函数如下所示：

```js
function anonymous() {
  with (this) {
    return _c('div', {
      staticClass: "child"
    }, [
      _t("header", [_c('div', [_v("Header")])]),
      _v(" "),
      _t("default", [_c('div', [_v("Main")])], {
        "message": message
      }),
      _v(" "),
      _t("footer", [_c('div', [_v("Footer")])])
    ], 2)
  }
}
```

## 总结

在编译的过程中，`Vue`会对`v-slot`指令和`<slot>`标签做不同的处理，在处理`v-slot`指令时，会将插槽的内容构建成一个个的函数，然后使用`_u`函数包裹，最后赋值给组件`AST`的`scopedSlots`，`<slot>`标签会使用`_t`函数包裹，内部会传入对应的插槽名称、后备内容、参数等。编译的结果都是为运行时服务的，那么在下一章节中，我们再详细分析在运行时中`Vue`是如何处理插槽的。
