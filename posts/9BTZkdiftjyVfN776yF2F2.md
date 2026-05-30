# Vue 中组合式函数模板引用导出出现未使用错误

使用组合式函数中经常会出现的写法，把一个模板的引用导出：

```ts
export const useSomething = () => {
  const containerRef = ref<HTMLElement | null>(null);
  // ...
  return {
    containerRef,
    ...
  }
};
```

然后在外面使用它：

```html
<script setup lang="ts">
const { containerRef } = useSomething();
</script>

<template>
  <div ref="containerRef"></div>
</template>
```

笨蛋智能感知是没法知道这个导出变量被使用，所以就开始大喊大叫了呢。

![alt text](/posts/imgs/fcwjpHKabVNTssj2e9PzYv.png)

## 解法

既然在内部声明导出到外部会被警告，那就直接在**外部声明**呢。

```ts
export const useSomething = (containerRef: Ref<HTMLElement | null>) => {
  // ...
  return {
    ...
  }
};
```

作为参数传进去就好了：

```html
<script setup lang="ts">
const containerRef = ref<HTMLElement | null>(null);
const { ... } = useSomething(containerRef);
</script>

<template>
  <div ref="containerRef"></div>
</template>
```

_斜体_
