> 控制手段不同、编译过程不同、编译条件不同

- 控制手段：`v-show` 隐藏则是为该元素添加 `css--display:none`，`dom` 元素依旧还在。`v-if` 显示隐藏是将 `dom` 元素整个添加或删除
- 编译过程：`v-if` 切换有一个局部编译/卸载的过程，切换过程中合适地销毁和重建内部的事件监听和子组件；`v-show` 只是简单的基于 `css` 切换
- 编译条件：`v-if` 是真正的条件渲染，它会确保在切换过程中条件块内的事件监听器和子组件适当地被销毁和重建。只有渲染条件为假时，并不做操作，直到为真才渲染`v-show` 由 `false` 变为 `true` 的时候不会触发组件的生命周期`v-if` 由 `false` 变为 `true` 的时候，触发组件的 `beforeCreate、create、beforeMount、mounted` 钩子，由 `true` 变为 `false` 的时候触发组件的 `beforeDestory、destoryed` 方法
- 性能消耗：`v-if` 有更高的切换消耗；`v-show` 有更高的初始渲染消耗；

#### 原理流程

**`v-show` 与 `v-if` 原理大致流程如下**

将模板 `template` 转为 `ast` 结构的 `JS` 对象
用 `ast` 得到的 `JS` 对象拼装 `render` 和 `staticRenderFns` 函数
`render` 和 `staticRenderFns` 函数被调用后生成虚拟 `VNODE` 节点，该节点包含创建 `DOM` 节点所需信息
`vm.patch` 函数通过虚拟 `DOM` 算法利用 `VNODE` 节点创建真实 `DOM` 节点

#### v-show 原理

不管初始条件是什么，元素总是会被渲染
我们看一下在 `vue` 中是如何实现的
代码很好理解，有 `transition` 就执行 `transition`，没有就直接设置 `display` 属性

```js
export const vShow = {
  beforeMount(el, { value }, { transition }) {
    el._vod = el.style.display === "none" ? "" : el.style.display;
    if (transition && value) {
      transition.beforeEnter(el);
    } else {
      setDisplay(el, value);
    }
  },
  mounted(el, { value }, { transition }) {
    if (transition && value) {
      transition.enter(el);
    }
  },
  updated(el, { value, oldValue }, { transition }) {
    // ...
  },
  beforeUnmount(el, { value }) {
    setDisplay(el, value);
  },
};
```

#### v-if 原理

`v-if` 在实现上比 `v-show` 要复杂的多，因为还有 `else` `else-if` 等条件需要处理，这里我们也只摘抄源码中处理 `v-if` 的一小部分

返回一个 `node` 节点，`render` 函数通过表达式的值来决定是否生成 `DOM`

```js
export const transformIf = createStructuralDirectiveTransform(
  /^(if|else|else-if)$/,
  (node, dir, context) => {
    return processIf(node, dir, context, (ifNode, branch, isRoot) => {
      // ...
      return () => {
        if (isRoot) {
          ifNode.codegenNode = createCodegenNodeForBranch(
            branch,
            key,
            context
          ) as IfConditionalExpression
        } else {
          // attach this branch's codegen node to the v-if root.
          const parentCondition = getParentCondition(ifNode.codegenNode!)
          parentCondition.alternate = createCodegenNodeForBranch(
            branch,
            key + ifNode.branches.length - 1,
            context
          )
        }
      }
    })
  }
)
```

#### v-show 与 v-if 的使用场景

- `v-if` 与 `v-show` 都能控制 `dom` 元素在页面的显示
- `v-if` 相比 `v-show` 开销更大的（直接操作 `dom` 节点增加与删除）
- 如果需要非常频繁地切换，则使用 `v-show` 较好
- 如果在运行时条件很少改变，则使用 `v-if` 较好
