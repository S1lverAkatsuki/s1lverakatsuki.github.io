# 前端背景视差动画

在著名斯拉夫大牢《逃离塔科夫》的主菜单里，有个背景视差动画。

<div align="center">
<img src="/posts/imgs/esmMwQSJssnXVJj1eU9BuX.gif" alt="alt text" style="zoom: 30%"/>
</div>

背景图片和前景的 UI 元素有个随鼠标移动而错位的效果。

我们来实现一下。

## 原理

拿到鼠标当前的坐标和整个背景（在这个场景里，是视口）的中心坐标算出差值，然后对横纵坐标的差值乘上一个系数作为背景的移动距离。以此实现视差。
整个流程挂到鼠标移动事件处理上，随鼠标移动更新背景的位移。

$$
背景位移量 = \left(鼠标坐标 - 中心点坐标\right) \times C
$$

## 具体实现

和我平时用的技术栈一样，Tailwind 写样式，Vue 做框架。换你自己的框架也不难。

```html
<script setup lang="ts">
import { onMounted, onUnmounted, ref } from "vue";

const appRootRef = ref<HTMLElement | null>(null);  // 视口
const parallaxLayerRef = ref<HTMLElement | null>(null);  // 要移动的背景层

const parallax = (e: MouseEvent) => {
  if (!appRootRef.value || !parallaxLayerRef.value) return;
  const rect = appRootRef.value.getBoundingClientRect();

  const centerX = rect.x + rect.width / 2;
  const centerY = rect.y + rect.height / 2;

  const mouseX = e.x;
  const mouseY = e.y;

  const C: number = 0.05;   // 缩放系数，想让画面跟着鼠标跑得猛一点就调大
  const deltaX = (centerX - mouseX) * C;
  const deltaY = (centerY - mouseY) * C;

  parallaxLayerRef.value.style.transform = `translate(${deltaX}px, ${deltaY}px)`;
};

onMounted(() => {
  window.addEventListener("mousemove", parallax);
});

onUnmounted(() => {
  window.removeEventListener("mousemove", parallax);
});
</script>

<template>
  <div
    ref="appRootRef"
    class="w-screen h-screen box-border relative overflow-hidden"
  >
    <!-- 背景层 -->
    <div class="absolute inset-0 overflow-hidden pointer-events-none">
      <div
        ref="parallaxLayerRef"
        class="absolute -inset-15 bg-img"
      ></div>
    </div>
    <!-- 内容层 -->
    <div class="relative w-full h-full flex items-center justify-center">
      <span class="text-2xl">中心点</span>
      ... 其他内容
    </div>
  </div>
</template>
```

效果如下：

<div align="center">
<img src="/posts/imgs/ea7DRvxfLL5PEwmkFMBcYC.gif" alt="alt text" style="zoom: 30%"/>
</div>

## 问题

如果位移缩放系数 `C` 改大了，`-inset-15` 就撑不住背景图不出界的范围了，鼠标一拖就会露出底下的白底。
那就别用固定 CSS 了，把 `-inset-15` 改成 `inset-0` 让背景单纯铺满视口，用 JS 来缩放：

```ts
// ...位移函数内部
parallaxLayerRef.value.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${1 + C})`; // 在这里用 scale 缩放背景图
```

## 性能优化

如果用户的鼠标回报率很高，`mousemove` 事件会触发得会非常频繁，不处理的话性能吃不消。
用 `requestAnimationFrame` 将整个动画部分包裹起来，让每一次重绘之前执行一次动画，频率和刷新率有关。

> 参考：[Window：requestAnimationFrame() 方法](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestAnimationFrame)

```ts
let ticking = false;  // 保证同一帧内只入队一次 rAF

const parallax = (e: MouseEvent) => {
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => {
    if (!appRootRef.value || !parallaxLayerRef.value) {
      ticking = false;
      return;
    }
    const rect = appRootRef.value.getBoundingClientRect();

    const centerX = rect.x + rect.width / 2;
    const centerY = rect.y + rect.height / 2;

    const mouseX = e.x;
    const mouseY = e.y;

    const C: number = 0.05;

    const deltaX = (centerX - mouseX) * C;
    const deltaY = (centerY - mouseY) * C;

    parallaxLayerRef.value.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${1 + C})`;

    ticking = false;  // 跑完一帧才重置
  });
};
```

## 缓动效果

如果把鼠标指针移出屏幕，那么会有一个很奇怪的现象：

<div align="center">
<img src="/posts/imgs/8dZskMd2eDggWNwu6CavkK.gif" alt="alt text" style="zoom: 30%"/>
</div>

虽然看起来不太自然，有个视差库（后面会说）做得挺好的：

<div align="center">
<img src="/posts/imgs/8h9W6jEB8YzVtqJ2bWYgk6.gif" alt="alt text" style="zoom: 30%"/>
</div>

如果我们只是对图片坐标做变换，没有做渐进动画的话，移出屏幕的瞬间画面一下就僵住了。
而样例就不一样，它有缓动追赶的效果。*准确来说，应该叫做「线性插值」吧*
所以要把目标偏移和当前偏移区分开，让背景追着鼠标跑。改动会比较大：

```ts
import { onMounted, onUnmounted, ref } from "vue";

const C: number = 0.5;
const EASE: number = 0.08; // 动画速度，调得越快越像瞬移

const appRootRef = ref<HTMLElement | null>(null);
const parallaxLayerRef = ref<HTMLElement | null>(null);

let targetX: number = 0;
let targetY: number = 0;
let currentX: number = 0;
let currentY: number = 0;
let animFrameId: number | null = null;  // requestAnimationFrame 自己就能返回 ID，也就不需要手动维护一个布尔值了

const parallax = (e: MouseEvent) => {
  if (!appRootRef.value) return;
  const rect = appRootRef.value.getBoundingClientRect();

  const centerX = rect.x + rect.width / 2;
  const centerY = rect.y + rect.height / 2;

  targetX = (centerX - e.x) * C;
  targetY = (centerY - e.y) * C;
};

const animate = () => {
  currentX += (targetX - currentX) * EASE;
  currentY += (targetY - currentY) * EASE;

  if (parallaxLayerRef.value) {
    parallaxLayerRef.value.style.transform = `translate(${currentX}px, ${currentY}px) scale(${1 + C})`;
  }

  animFrameId = requestAnimationFrame(animate);
};

onMounted(() => {
  window.addEventListener("mousemove", parallax);
  animFrameId = requestAnimationFrame(animate);
  // 分开了更新当前偏移的逻辑和计算目标偏移的逻辑
});

onUnmounted(() => {
  window.removeEventListener("mousemove", parallax);
  if (animFrameId !== null) {
    cancelAnimationFrame(animFrameId);
    animFrameId = null;
  }
});
```

## 用现成的库

有个库叫 [parallax.js](https://github.com/wagerfield/parallax)，上面这些东西它直接就能实现。
而且还支持分层视差，不同层位移幅度不一样，不只是背景动一下那么简单。
真要做东西的话还是用它省事。
*果然这就是 JS 生态的终点啊，一切转导包*
