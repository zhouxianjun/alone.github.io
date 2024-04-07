---
title: useDialog hook 封装 el-dialog
date: 2024-04-06 21:08:19
tags:
  - vue
  - vue3
  - element-ui
  - element-plus
  - hook
  - el-dialog

categories:
- 前端
- vue
---


在前端开发中，弹窗是一个常见的需求，而Element UI框架中的el-dialog组件提供了弹窗的基本功能。然而，在实际开发中，我们可能会遇到一些需要定制化的需求，
比如需要对弹窗进行二次封装，以便在整个项目中统一管理弹窗的样式和行为。在这篇文章中，我将分享如何使用useDialog Hook来封装el-dialog，以实现更灵活、更易用的弹窗组件。


## 问题

在实际开发过程中，我们经常遇到一个场景：一个通用的购买组件需要在多个页面中使用。例如，在一个订阅服务的应用程序中，用户可能在订阅页面直接进行购买操作，同时，在其他页面浏览不同的服务时，根据业务逻辑判断是否需要购买，则需弹出同样的购买对话框提示用户。

为了实现这一功能，我们通常会采取以下步骤：

封装购买组件：首先创建一个通用的购买组件，以便在不同的页面和场景下复用。
在订阅页面渲染购买组件：将购买组件直接嵌入到订阅页面中。
在其他页面使用el-dialog展示购买组件：在不同的功能页面上，通过el-dialog来控制购买组件的显示，利用一个visible状态变量（通常是一个ref响应式变量）来动态控制对话框的弹出与关闭。
以上实现方式虽然可以满足功能需求，但随着该购买组件被越来越多的页面和功能所使用，维护起来就变得越加复杂和繁琐。我们必须在每个需要使用该组件的页面中重复编写控制显示隐藏的逻辑代码。

因此，一个明显的问题出现了：有没有更好的方法来简化这个过程？是否可以通过某种方式，使用一个单独的函数来全局控制购买组件的打开或关闭，从而减少代码重复并降低维护成本呢？


## 什么是useDialog Hook？

在Vue中，Hook是一种让你在函数式组件或者API中“钩入”Vue特性的方式。它们通常在合成API(`Composition API`)中使用，这是Vue提供的一套响应式和可复用逻辑功能的集合。
例如，`useDialog` Hook可能就是一个封装了`<el-dialog>`组件基本功能的自定义Hook，并可能还提供了附加的特性以便更方便地在项目中管理&展示弹窗。


## useDialog Hook的实现

我们需要达到的目标：

1. 能够满足基础用法，传入`el-dialog`的基础属性以及默认slot显示的内容，导出 `openDialog` 和 `closeDialog·`函数
2. 支持`el-dialog`的事件配置
3. 支持默认slot组件的属性配置
4. 支持`el-dialog`其他`slot`配置，如`header`和`footer`等
5. 在内容组件中抛出特定事件支持关闭dialog
6. 支持显示内容为`jsx`、`普通文本`、`Vue Component`
7. 支持在显示内容中控制是否可以关闭的回调函数，例如`beforeClose`
8. 支持显示之前钩子例如`onBeforeOpen`
9. 支持定义和弹出时修改配置属性
10. 支持继承root vue的prototype，可以使用例如`vue-i18n`的`$t`函数
11. 支持`ts`参数提示


### 准备`useDialog.ts`文件实现类型定义

```ts
import type { Ref } from 'vue'
import { h, render } from 'vue'
import { ElDialog } from 'element-plus'
import type {
  ComponentInternalInstance,
} from '@vue/runtime-core'

type Content = Parameters<typeof h>[0] | string | JSX.Element
// 使用 InstanceType 获取 ElDialog 组件实例的类型
type ElDialogInstance = InstanceType<typeof ElDialog>

// 从组件实例中提取 Props 类型
type DialogProps = ElDialogInstance['$props'] & {
}
interface ElDialogSlots {
  header?: (...args: any[]) => Content
  footer?: (...args: any[]) => Content
}
interface Options<P> {
  dialogProps?: DialogProps
  dialogSlots?: ElDialogSlots
  contentProps?: P
}
```

### 接着实现普通`useDialog`函数

下面函数我们实现了基础的用法，包括了：1，2，3，4，6，11 目标

```ts
export function useDialog<P = any>(content: Content, options?: Ref<Options<P>> | Options<P>) {
  let dialogInstance: ComponentInternalInstance | null = null
  let fragment: Element | null = null

  // 关闭并卸载组件
  const closeAfter = () => {
    if (fragment) {
      render(null, fragment as unknown as Element) // 卸载组件
      fragment.textContent = '' // 清空文档片段
      fragment = null
    }
    dialogInstance = null
  }
  function closeDialog() {
    if (dialogInstance)
      dialogInstance.props.modelValue = false
  }

  // 创建并挂载组件
  function openDialog() {
    if (dialogInstance) {
      closeDialog()
      closeAfter()
    }
    
    const { dialogProps, contentProps } = options
    fragment = document.createDocumentFragment() as unknown as Element

    const vNode = h(ElDialog, {
      ...dialogProps,
      modelValue: true,
      onClosed: () => {
        dialogProps?.onClosed?.()
        closeAfter()
      },
    }, {
      default: () => [typeof content === 'string'
        ? content
        : h(content as any, {
          ...contentProps,
        })],
      ...options.dialogSlots,
    })
    render(vNode, fragment)
    dialogInstance = vNode.component
    document.body.appendChild(fragment)
  }

  onUnmounted(() => {
    closeDialog()
  })

  return { openDialog, closeDialog }
}
```

### 接下来我们实现目标`5`

1. 需要在定义中支持`closeEventName`

```ts
interface Options<P> {
  // ...
  closeEventName?: string // 新增的属性
}
```

2. 修改`useDialog`函数接收`closeEventName`事件关闭`dialog`

```ts
export function useDialog<P = any>(content: Content, options?: Ref<Options<P>> | Options<P>) {
  // ...
  // 创建并挂载组件
  function openDialog() {
    // ...
    fragment = document.createDocumentFragment() as unknown as Element
    // 转换closeEventName事件
    const closeEventName = `on${upperFirst(_options?.closeEventName || 'closeDialog')}`

    const vNode = h(ElDialog, {
      // ...
    }, {
      default: () => [typeof content === 'string'
        ? content
        : h(content as any, {
          ...contentProps,
          [closeEventName]: closeDialog, // 监听自定义关闭事件，并执行关闭
        })],
      ...options.dialogSlots,
    })
    render(vNode, fragment)
    dialogInstance = vNode.component
    document.body.appendChild(fragment)
  }

  onUnmounted(() => {
    closeDialog()
  })

  return { openDialog, closeDialog }
}
```

### 接下来我们实现目标`7`、`8`

1. 需要在定义中支持 `onBeforeOpen`， `beforeCloseDialog` 默认是传给内容组件，有组件调用设置

```ts
type DialogProps = ElDialogInstance['$props'] & {
  onBeforeOpen?: () => boolean | void
}
```

2. 修改`useDialog`函数接收`onBeforeOpen`事件&传递`beforeCloseDialog`

```ts
export function useDialog<P = any>(content: Content, options?: Ref<Options<P>> | Options<P>) {
  // ...
  // 创建并挂载组件
  function openDialog() {
    // ...
    const { dialogProps, contentProps } = options
    // 调用before钩子，如果为false则不打开
    if (dialogProps?.onBeforeOpen?.() === false) {
      return
    }
    // ...
    // 定义当前块关闭前钩子变量
    let onBeforeClose: (() => Promise<boolean | void> | boolean | void) | null

    const vNode = h(ElDialog, {
      // ...
      beforeClose: async (done) => {
        // 配置`el-dialog`的关闭回调钩子函数
        const result = await onBeforeClose?.()
        if (result === false) {
          return
        }
        done()
      },
      onClosed: () => {
        dialogProps?.onClosed?.()
        closeAfter()
        // 关闭后回收当前变量
        onBeforeClose = null
      },
    }, {
      default: () => [typeof content === 'string'
        ? content
        : h(content as any, {
          // ...
          beforeCloseDialog: (fn: (() => boolean | void)) => {
            // 把`beforeCloseDialog`传递给`content`，当组件内部使用`props.beforeCloseDialog(fn)`时，会把fn传递给`onBeforeClose`
            onBeforeClose = fn
          },
        })],
      ...options.dialogSlots,
    })
    render(vNode, fragment)
    dialogInstance = vNode.component
    document.body.appendChild(fragment)
  }

  onUnmounted(() => {
    closeDialog()
  })

  return { openDialog, closeDialog }
}
```

### 接下来我们实现目标`9`、`10`

```ts
// 定义工具函数，获取计算属性的option
function getOptions<P>(options?: Ref<Options<P>> | Options<P>) {
  if (!options)
    return {}
  return isRef(options) ? options.value : options
}

export function useDialog<P = any>(content: Content, options?: Ref<Options<P>> | Options<P>) {
  // ...
  // 获取当前组件实例，用于设置当前dialog的上下文，继承prototype
  const instance = getCurrentInstance()
  // 创建并挂载组件，新增`modifyOptions`参数
  function openDialog(modifyOptions?: Partial<Options<P>>) {
    // ...
    const _options = getOptions(options)
    // 如果有修改，则合并options。替换之前的options变量为 _options
    if (modifyOptions)
      merge(_options, modifyOptions)
    
    // ...

    const vNode = h(ElDialog, {
      // ...
    }, {
      // ...
    })
    // 设置当前的上下文为使用者的上下文
    vNode.appContext = instance?.appContext || null
    render(vNode, fragment)
    dialogInstance = vNode.component
    document.body.appendChild(fragment)
  }

  onUnmounted(() => {
    closeDialog()
  })

  return { openDialog, closeDialog }
}
```


通过上面的封装，我们可以看到使用useDialog Hook之后，我们只需要在需要弹窗的地方引入这个Hook，然后调用openDialog方法即可，非常方便和简洁。
而且这样的封装还可以让我们在以后需要修改弹窗逻辑的时候更加方便，只需要在useDialog Hook中进行修改即可，不需要在各个地方进行重复的修改。

## 使用我们的`useDialog` hook 解决我们上面的问题

### 创建`components/buy.vue`购买组件
```vue
<script lang="ts" setup>
  const props = defineProps({
    from: {
      type: String,
      default: '',
    },
  })
</script>
<template>
  我是购买组件
</template>
```

### 在`pages/subscription.vue`页面中使用`buy.vue`购买组件
```vue
<script lang="ts" setup>
  import Buy from '@/components/buy.vue'
</script>
<template>
<!-- ... -->
  <Buy from="subscription" />
<!-- ... -->
</template>
```

### 在其他功能页面中弹出`buy.vue`购买组件
```vue
<script lang="ts" setup>
  import { useDialog } from '@/hooks/useDialog'
  const Buy = defineAsyncComponent(() => import('@/components/buy.vue'))

  const { openDialog } = useDialog(Buy, {
    dialogProps: {
      // ...
      title: '购买'
    },
    contentProps: {
      from: 'function',
    },
  })
  
  const onSomeClick = () => {
    openDialog()
  }
</script>
```

## 其他useDialog hook的应用

### `beforeClose` & `closeEventName` 示例

`buy.vue`购买组件

```vue
<script lang="ts" setup>
  const props = defineProps({
    from: {
      type: String,
      default: '',
    },
    beforeCloseDialog: {
      type: Function,
      default: () => true,
    },
  })
  
  const emit = defineEmits(['closeDialog'])

  props.beforeCloseDialog(() => {
    // 假如from 为 空字符串不能关闭
    if (!props.from) {
      return false
    }
    return true
  })
  
  // 关闭dialog
  const onBuySuccess = () => emit('closeDialog')
</script>
```
```vue
<script lang="ts" setup>
  import { useDialog } from '@/hooks/useDialog'
  const Buy = defineAsyncComponent(() => import('@/components/buy.vue'))

  const { openDialog } = useDialog(Buy, {
    dialogProps: {
      // ...
      title: '购买'
    },
    contentProps: {
      from: '',
    },
  })
  
  const onSomeClick = () => {
    openDialog()
  }
</script>
```


## 总结
通过使用useDialog Hook封装el-dialog，我们可以让前端技术变得更加有趣和简洁。希望大家也能尝试一下这样的封装方式，让我们的前端代码变得更加优雅和易维护。就像一位优秀的厨师一样，掌握了精妙的调味技巧，让每道菜都变得美味可口！