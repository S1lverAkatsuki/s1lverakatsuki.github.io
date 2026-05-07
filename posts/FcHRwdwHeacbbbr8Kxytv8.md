# 博客开发笔记

*历经两年之久的整理*
（立项的时候是 2024 年，直到最近才完成了编写）

> 2026-05-04：新增章节「Web Storage API `sessionStorage` 存储网站信息」

## 一种通用的 composables 组合式函数结构

```ts
const doSomething = () => {
  // ...
};

export const useXXX = () => {
  const foo = ref(42);

  return {
    foo,
    doSomething
  };
}
```

只暴露出一个函数接口，用的时候解出来：

```ts
const { foo, doSomething } = useXXX();
```

当然也可以只用一个：

```ts
const { doSomething } = useXXX();
// 确保变量名与组合式函数中的返回值一致，不然会 undefined
```

> 有关 JS 解构语法参照：[ES6 解构赋值](https://www.runoob.com/w3cnote/deconstruction-assignment.html)

## 很好的 JS 文件路径管理

> 见 `/scrips` 内的所有脚本

```ts
const ROOT_DIR = path.join("root");   // /root
const IMG_DIR = path.join(ROOT_DIR, "img");  // /root/img
const SUB_FOLDER_DIR = path.join(ROOT_DIR, "folder");  // root/folder
```

用内置方法管理路径，比硬搓字符串稳定多了。
不同系统上对于斜杠的处理不同（`/` 和 `\`），拼字符串不能跨平台。

> 参照：[Node.js path 模块](https://www.runoob.com/nodejs/nodejs-path-module.html)
> *Node 中文网的文档字全和背景混一块了，真抽象*

---

当然，如果需要自己输入路径，那就用 `path.normalize` / `path.resolve` 进行规范化，去掉相对路径这种有歧义的，转成内置类型。

## 不定量参数

```ts
const f = (...args: unknown[]) => {};
```

可以塞进任意数量的参数，记得类型是数组。

## AnyScript 与 UnknownScript

众所周知，`any` 类型代表这个变量不会被类型限制约束，可以赋给任何值，也可也接受任何值。
`unknown` 类型有一点点限制：它不能被赋给除了同种类型和 `any` 的变量，也不能调用它的方法。
如果有些类型不确定的时候，现在都可以无脑替换成 `unknown` 类型。
*难怪 ESLint 把 any 禁掉了，原来有更优的解法*

---

用 `unknown` 的时候，得用类型断言确认其是否是想要的类型。
*不过感觉用的时候大多数是不清楚类型（比如偷懒的 JS 库）的时候*

## switch-case 多分支合并

```ts
switch (fruit) {
  case "Apple":
  case "Pear":
    // ...
    break;
  case "Orange":
    // ...
    break;
}
```

苹果和梨就都会走下面那条分支。
*话说为什么这么基础的语法都能进这个榜单*

## 基于路由的组件结构

在 `App.vue` 里放的是 `<router-view />`，然后编写的就都是 `views` 文件夹下面的视图，对应不同页面。

## 箭头函数的泛型写法

和其他语言里不太一样，记一下。

```ts
const f = <T>(foo: T): T => {
  // ...
  return foo;
};
```

## v-for 中的 `:key` 属性

拿来标识这个 DOM 的唯一性，方便系统进行更新，在一些经常变化的场合，可以靠 key 来让系统减少更新，直接复用旧 DOM。
一般是索引，id 之类的。

## JS 中的深浅拷贝

所有赋值全是浅拷贝。实现深拷贝，得新开对象。

```ts
let user = { name: "John" };
let user1 = {};

Object.assign(user, user1);
// 或者是用 spread 语法
// user1 = {...user};
```

这两个原生语法，对于属性值是基本类型（`number`，`string`）的属性执行深拷贝，而对于属性值是对象或其他引用类型的属性执行浅拷贝。
如果真要有类似 C 系语言的深拷贝语义，那就导包吧 [__cloneDeep](https://lodash.com/docs/4.17.15#cloneDeep)

---

这是关于默认值 `src/views/Articles.vue` 中有两种给参数赋默认值的处理。一种是形如`{...user}` 这样的开对象。
还有另一种关于参数默认值的写法，具体在 `src/types/filterRule.ts` 中有实现，靠工厂函数直接开新对象。

```ts
interface User {
  name: string
};

const createDefault = (): User => ({
  name: "foo"
});

const user = createDefault();
```

## 正则毒点

出现在对输入的处理上，如果输入一点特殊的正则字符，比如 `\w` 之类的就炸了。
用下面的转义函数吧。

```ts
const escapeRegExp = (value: string) => value.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
```

输入的 `\w` 会被转义成 `\\w`

## `localCompare` 方法

```ts
"a".localCompare("b")  // -1
```

比较字符串的字典序，会按照不同语言的顺序进行比较。
在 `src/views/Articles.vue` 中，拿来进行日期字符串（比如`2025-11-07`）的比较。
用的时候记得那个经典的问题：`"2" > "10"`。

## `<script setup>`

由于没有使用模板自动生成 `.vue` 组件，所以有个组件内我漏加了 `setup` 修饰。发现 `import` 导入的组件不能在模板中使用。

> 参照：[\<script setup\>](https://cn.vuejs.org/api/sfc-script-setup.html)
> 任何在 \<script setup\> 声明的顶层的绑定 (包括变量，函数声明，以及 import 导入的内容) 都能在模板中直接使用

记得加就好。

## `watch` 与 `watchEffect`

> 参照：[侦听器](https://cn.vuejs.org/guide/essentials/watchers)

我在用 watch 监听一个嵌套元素的变化的时候发现不能正确触发更新，因为被监听对象没有被替换。

使用诸如下面这种方式时，想要监听深层对象得加`{ deep: true }`

```ts
watch(
  () => state.someObject,
  (value) => {
    // ...
  }
  { deep: true }
);
```

## 动态计算字符 宽度

> 见 `src\components\search\filters\tag\TagMultiSelect.vue`

和注释里说的一样，那个 `style="{ position: 'absolute', left: '-9999px' }"` 的组件就是为了把字塞进去，然后直接获取它的宽高（函数 `updateVisibleSelections()`），免去了复杂不稳定的计算。

## `ref` 引用组件

用这个模板当例子：

```html
<template>
  <div class="foo">随便一个 DOM</div>
</template>
```

在原生 JS 里，是靠元素选择器抓元素的：

```ts
const foo = document.querySelector(".foo");

if (!foo) return;

foo.innerHTML = "用抓到的 DOM 做点什么";
```

现代 Vue 提供了一种更优雅的方式：`ref`。
只需要给模板里加一个标签。

```html
<template>
  <div ref="fooRef" class="foo">随便一个 DOM</div>
</template>
```

`fooRef` 是一个需要声明的响应式变量。

```ts
const fooRef = ref<HTMLElement | null>(null);

if (!fooRef.value) return;

const item = fooRef.value;  // 复杂操作一般解出来用，少了几个.value
item.innerHTML = "优雅";
```

## 字典统计次数的演化

总会有使用字典统计一个数组（或者是字符串等等）中元素出现次数的情景，没有方法就得手写。

```ts
const arr = ["a", "b", "c", "b", "d", "c", "c"];

const countMap = new Map<string, number>();

arr.forEach(element => {
  if (countMap.has(element)) {
    countMap.set(element, countMap.get(element) + 1);
  } else {
    countMap.set(element, 1);
  }
});
```

很 C++ 风格（贬义）的写法，还会在调用 `get` 的时候提示你可能为 `undefined`（因为笨蛋类型检查不认）。

用一下 `??` 或是 `||` 语法可以把 if 语句省掉。
如果没有键，`get` 会返回 `undefined`，符号会选择返回另一个 `0` 。

```ts
arr.forEach(element => {
  countMap.set(element, (countMap.get(element) ?? 0) + 1);
  // 注意运算优先级，?? 和 || 优先级比加减乘除低，得套括号
});
```

## 推断 JSON 文件类型

这里有一个会在哪里都可能见到类似的 JSON 文本（文件名是 `user.json`）。

```json
{ 
  "user": {
    "name": "Foo",
    "age": 24,
    "id": [
      "aaa",
      "bbb"
    ]
  } 
}
```

类型都可以推出来：

```ts
import userJson from "user.json";

type RawUserJson = typeof userJson;
type User = RawUserJson["user"];

const user: User = userJson.user;
```

## Web Storage API `sessionStorage` 存储网站信息

*我们不做小饼干*
负责处理刷新后过滤器重置的问题，只要拿一个状态存起来就好了嘛。
这是个比较新的 API，不过据 MDN 说支持的挺广的......吧。

> 参考：[Web Storage API](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Storage_API)

用到的是 `sessionStorage`，它会在页面被关闭的时候自动清空，很适合这种随用随关的场景。
而且没有隐私风险，也就意味着不需要询问用户是否同意保存，省事。

这两个封装的函数很简单：

```ts
const loadFromStorage = <T, >(key: string, fallback: T): T => {
  try {
    const stored = sessionStorage.getItem(key);
    return stored ? (JSON.parse(stored) as T) : fallback;
  } catch {
    return fallback;
  }
};

const saveToStorage = <T, >(key: string, value: T): void => {
  sessionStorage.setItem(key, JSON.stringify(value));
};
```

载入函数提供了回退项，以免载入失败 / 没有存储的时候出来个 `undefined` 恶心人。
用起来也是一样的简单。

```ts
type User = {
  name: string,
  age: number
};

const user1: User = { name: "Yukari", age: 17 };

saveToStorage<User>("user", user1);

const loaded = loadeFromStorage<User>("user", { name: "aaa", age: 0 });
```
