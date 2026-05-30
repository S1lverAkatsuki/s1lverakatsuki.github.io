# 前端背景视差动画

在著名斯拉夫大牢《逃离塔科夫》中，游戏外菜单有个背景视差动画。

<div align="center">
<img src="img/screenshots.gif" alt="alt text" style="zoom: 30%"/>
</div>

背景图片和前景的 UI 元素有个随鼠标移动而错位的效果。

我们来实现这点。

## 原理

拿到鼠标当前的坐标和整个背景（在这个场景里，是视口）的中心坐标算出差值，然后对横纵坐标的差值乘上一个系数作为背景的移动距离。以此实现视差。
整个流程挂到鼠标移动事件处理上，随鼠标移动更新背景的位移。

$$
背景位移量 = \left(鼠标坐标 - 中心点坐标\right) \times C
$$

## 具体实现

和我经常用的技术栈一样，用的 Tailwind 来方便写样式，Vue 做框架。
*我相信你能做得到迁移到你用的框架的！*

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

  const C: number = 0.05;   // 缩放系数，想要移动幅度更猛烈的话就调大
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
<img src="img/screenshots1.gif" alt="alt text" style="zoom: 30%"/>
</div>

## 问题

如果位移缩放系数 `C` 改大了，那么原有的 `-inset-15` 就不能完全将背景图缩放到不会出界的范围，随着鼠标拖动就会露出背景图下的白底。
那就不用固定的 CSS 样式处理好了，去除样式中的 `-inset-15` 改为 `inset-0` 单纯占满整个视口，然后用 JS 缩放：

```ts
// ...位移函数内部
parallaxLayerRef.value.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${1 + C})`; // 在这里用 scale 缩放背景图
```

## 性能优化

如果用网站的人是个电竞小子，拿着回报率超高的鼠标，频繁更新网站会卡爆的。
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

    ticking = false;  // 跑完动画才重置回可用状态
  });
};
```

## 缓动效果

如果把鼠标指针移出屏幕，那么会有一个很奇怪的现象：

<div align="center">
<img src="img/screenshots2.gif" alt="alt text" style="zoom: 30%"/>
</div>

虽然听起来很令人郁闷，但是他们真的做的很好：

<div align="center">
<img src="img/screenshots3.gif" alt="alt text" style="zoom: 30%"/>
</div>

因为我们的只是对图片本身的坐标进行变换，并没有做什么渐进的动画。
将目标偏移和当前偏移区分开来就可以写出追赶的动画。
这需要进行一次超级大的改动欸：

```ts
import { onMounted, onUnmounted, ref } from "vue";

const C: number = 0.5;
const EAST: number = 0.08; // 这是动画速度，调的越快越接近瞬移

const appRootRef = ref<HTMLElement | null>(null);
const parallaxLayerRef = ref<HTMLElement | null>(null);

let targetX = 0;
let targetY = 0;
let currentX = 0;
let currentY = 0;
let animFrameId: number | null = null;  // 取代了 track，因为好像 requestAnimationFrame 能直接返回一个 ID，不需要额外弄一个布尔值来标记

const parallax = (e: MouseEvent) => {
  if (!appRootRef.value) return;
  const rect = appRootRef.value.getBoundingClientRect();

  const centerX = rect.x + rect.width / 2;
  const centerY = rect.y + rect.height / 2;

  targetX = (centerX - e.x) * C;
  targetY = (centerY - e.y) * C;
};

const animate = () => {
  currentX += (targetX - currentX) * EAST;
  currentY += (targetY - currentY) * EAST;

  if (parallaxLayerRef.value) {
    parallaxLayerRef.value.style.transform = `translate(${currentX}px, ${currentY}px) scale(${1 + C})`;
  }

  animFrameId = requestAnimationFrame(animate);
};

onMounted(() => {
  window.addEventListener("mousemove", parallax);
  animFrameId = requestAnimationFrame(animate);
  // 把更新当前偏移量的函数同更新鼠标坐标计算目标偏移量的函数分开了
  // 同样，动画对象也不会一直创建了，只在页面生命周期结束的时候才销毁
  // 如果目标偏移量同当前偏移量相等也没有动画的事，一旦鼠标动了更新了目标偏移量那动画才有效果
});

onUnmounted(() => {
  window.removeEventListener("mousemove", parallax);
  if (animFrameId !== null) {
    cancelAnimationFrame(animFrameId);
    animFrameId = null;
  }
});
```

## 用库逃课

*果然还是 JS 生态的终点吗，还是不如现成的轮子呢*
有个库叫做 [parallax.js](https://github.com/wagerfield/parallax)，它可以轻松实现这里面的功能。
而且支持分层视差，不同层的位移幅度不同，而不仅仅是背景的相对位移。
如果我去实际做东西肯定用这个库。
