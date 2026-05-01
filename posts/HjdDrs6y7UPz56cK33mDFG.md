# 使用 vuedraggable 做列表元素拖动换位

最近在重写 LLM-Prompt-Manager 的时候，我得写一个拖曳改变 PromptItem 的排列顺序的功能，然后我就用了一下 [vuedraggable](https://github.com/SortableJS/Vue.Draggable) 这个库，整理了一些用法记录在这里。

## 最简单的样例

```vue
<script setup>
import { ref, watchEffect } from "vue";
import draggable from "vuedraggable";

const list = ref([
  { id: 1, name: "苹果" },
  { id: 2, name: "香蕉" },
  { id: 3, name: "橙子" },
]);

watchEffect(() => console.log(list.value));
</script>

<template>
  <div class="w-full">
    <div class="w-[30%] pt-20 ml-auto mr-auto">
      <draggable
        handle=".drag-handler"
        v-model="list"
        item-key="id"
        class="w-full flex flex-col gap-2"
      >
        <template #item="{ element }">
          <div class="w-full p-2.5 border border-zinc-500 bg-gray-100">
            <span class="select-none drag-handler">
              ⠿
            </span>
            {{ element.name }}
          </div>
        </template>
      </draggable>
    </div>
  </div>
</template>
```

效果如下，非常简单：

![alt text](/posts/imgs/ePv6Y9WYsQ7unP7VDXhjH9.gif)

---

要想使用只需要用 `<draggable>` 标签替换原来的 `<div v-for="...">` 即可。
记得必须提供一个 `v-model` 作为信息来源与更改的数组对象，不然无法进行渲染。
`<draggable>` 标签本身只能含有一个子元素 `<template>`，传递的 `element` 即为 `v-model` 传入的数组的单个元素。
在 `<template>` 内塞要拖动的单个元素的模板。

<br/>

`<draggable>` 标签的 `handle` 属性为模板内哪个类可以触发拖动事件的触发器，`handle=".aaa"` 也就是含有 `aaa` 类的 DOM 可以触发拖动事件。
这里是 `span.drag-handler` DOM。*里面塞的元素居然是骨牌的六点*

## 更好看的样式

![alt text](/posts/imgs/r5FNnZKrUzun1sqrzK4PSh.gif)

这个效果如何，没有了难看的系统样式，有了一个占位符来表明实际插入的位置。

多了这些内容：

```vue
<template>
  <div>
    <div>
      <draggable
        :force-fallback="true"
        fallback-class="drag-fallback"
        :fallback-on-body="true"
        ghost-class="drag-ghost"
      >
        <template>
          <div
            draggable="false"
          >
          </div>
        </template>
      </draggable>
    </div>
  </div>
</template>

<style scoped lang="css">
.drag-ghost {
  opacity: 0.4;
  border: 2px dashed #94a3b8 !important;
  background: #f1f5f9 !important;
}

:global(.drag-fallback) {
  opacity: 1 !important;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
}
</style>
```

我删去了没必要的部分避免干扰，反正 DOM 树是一模一样的，你也能对着看。

<br/>

`:force-fallback` 是让库选择自己复制 DOM 而不是使用原生的 HTML5 API 的 drag image 实现跟随鼠标移动的提示 DOM。
`fallback-class` 就是对跟随鼠标的元素附加的样式，这里面的 `!important` 都是没法删的，不然会被默认样式顶掉。

**Tip**：
`:global(...)` 作用是消除了 `scoped`导致的 Vue 编译的样式的作用域限制。
比如说：

```html
<style scoped lang="css">
.aaa {
  color: red
}
</style>
```

Vue 会自动给组件模板里的每个 HTML 元素加上一个独特的 data 属性（比如 `data-v-abc123` ），并在 CSS 选择器末尾加上对应的属性选择器。这样样式就只作用于当前组件，不会泄露到外部。
这里会在编译的时候被转为 `.aaa[data-v-abc123]`。

加上 `:global()` 后就没有这种处理步骤。

当然也可以直接使用 `<style>` 标签规避掉这点。

---

`fallback-on-body` 是指让复制出的随鼠标移动的元素固定在 `<body>` 层级而不是加到拖拽起始位置的父容器内部末尾，也就是 `<draggable>` 的根元素里面。
如果父容器有 `overflow: hidden` 或裁剪相关样式，可能会被截断。
加上 `fallback-on-body` 后，克隆元素会直接挂到 `<body>` 下，并用 `position: fixed` 跟随鼠标。
一般情况下推荐开上就是了。

## 没有幽灵元素的版本

所谓的「幽灵元素」，就是这里的 `.drag-ghost`，被拖曳的元素在列表本身的占位符。它的位置决定了停止拖曳后被拖曳的元素停靠的位置。
只要让它**不可见**就能实现这种效果：

![alt text](/posts/imgs/dSwNWFyWbxmCgA5ov3trrf.gif)

很显然，这确实不是真正的「没有」呢。因为库的计算要用。
靠改透明度实现不可见，`display: none` 会让复制出来的元素错误：

```css
.drag-ghost {
  opacity: 0;
}
```
