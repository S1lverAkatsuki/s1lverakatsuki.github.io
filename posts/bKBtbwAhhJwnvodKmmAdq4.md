# hd2-random 开发笔记

[仓库地址](https://github.com/S1lverAkatsuki/hd2-random)

## 利用 blob 实现图片缓存

blob 是一个二进制文件，拿来缓存资源用的，除了图片，音视频文字都可以。
在 `useImageCache.ts` 内。
通过 `url2Blob` 来获取一个资源的 blob：

```ts
const url2Blob = async (url: string): Promise<Blob> => {
  return window.fetch(url).then(response => response.blob());
};
```

通过对 `keyUrlDict` 这个字典的维护，可以让 blob 资源不被浏览器释放掉。  
让它一直在浏览器中运行，就能在第一次加载的时候加载所有资源，后面就不会每次加载都卡一下。

## 项目中的 composables 文件夹

> 参考：[组合式函数](https://cn.vuejs.org/guide/reusability/composables)

响应式函数，负责解耦 `<script>` 标签，可以维护一个状态。  
命名规则为 `use` + `CamelCase`，例如 `useEvenListener`：

```ts
import { onMounted, onUnmounted } from "vue";

export function useEventListener(target: any, event: any, callback: any) {
  // 好孩子别学 anyscrpit
  onMounted(() => target.addEventListener(event, callback))
  onUnmounted(() => target.removeEventListener(event, callback))
}
```

在组件内就可以当作是原有的函数去使用。  
每个调用组合式函数的组件状态都是独立的，别怕会互相干扰。

## 实现网页的多语言

### vue-i18n

1. 导入
  vue-i18n 算是 Vue 本体的插件了

  ```bash
  pnpm i vue-i18n
  ```

2. 配置插件
  （2. 3. 条都在 `src` 文件内更改，这里也设置了`@`的替换）

  `/plugins/i18n.ts`

  ```ts
  import { createI18n } from "vue-i18n";

  import { messages } from "@/locales";

  // 支持的语言列表
  const SUPPORTED_LOCALES = ["zh", "en"];

  /**
  * 检测浏览器语言，默认 fallback 为英文
  */
  function detectLocale(): string {
    // 获取浏览器语言
    const browserLang = navigator.language || navigator.languages?.[0] || "en";

    // 匹配语言代码（如 zh-CN -> zh）
    const langCode = browserLang.split("-")[0];

    // 如果支持该语言则返回，否则返回英文
    // 调试用这个
    // return "en";
    return SUPPORTED_LOCALES.includes(langCode) ? langCode : "en";
  }

  /**
  * 创建并配置 i18n 实例
  */
  export const i18n = createI18n({
    legacy: false,
    locale: detectLocale(),
    fallbackLocale: "en",
    messages
  });
  ```

3. 写语言文件
  上面定义了`SUPPORTED_LOCALES`这个数组，也记得提供相同的语言文件数量

  `/locals/zh.ts`

  ```ts
  const zh = {
    foo: "为什么",
    bar: {
      baz: "秦始皇没干完活啊",
    }
  }
  ```

  英文同理

4. 应用
    - 在组件内导入

    ```ts
    import { useI18n } from "vue-i18n";

    const { t, locale } = useI18n();
    const currLang = locale.value;  // 这就是当前语言类型，对应 SUPPORTED_LOCALES
    ```

    - 这样在 DOM 模板内，就能使用了

    ```html
    <div>{{ t("foo") }}</div>
    <!-- 为什么 -->
    <div>{{ t("bar.baz") }}</div>
    <!-- 秦始皇没干完活啊 -->
    ```

### 不同语言，不同样式

可以使用 `:class` 语法给主容器加一个

```html
<div :class="[currLang + '-style']" > ... </div>
```

`currLang` 参照上文，是声明的语言类型
这样就有两个容器选择器了

- `div.zh-style`
- `div.en-style`

然后就可据此套娃选择器了，比如：

```css
div.zh-style {
  /* 这里装中文样式 */
}

div.en-style {
  /* 这里装英文样式 */
}
```

## defineModel 默认值

```ts
const model = defineModel<string>({ default: "" });
```

## 包管理器 pnpm

*本征力推*

### 为什么要更换？

- pnpm 和 npm 几乎完全兼容
- pnpm 比 npm 快两倍
- pnpm 的软连接让 node_modules 显著降低体积
- pnpm 可以简写命令（e.g. `npm run dev` 写成 `pnpm dev`）

### 如何从 npm 迁移到 pnpm

1. ~~ntr 正主~~

  ```bash
  npm i -g pnpm
  ```

2. 删掉 `node_modules` 和 `package-lock.json`
3. 让 pnpm 补全

  ```bash
  pnpm i
  ```

## template 内 class 多标签

class 内可以用空格分割多个参数，这是常识，对吧。

```html
<div class="foo bar" />
```

它会被下面这几种选择器选择：

```css
.foo {
  /* 占位置 */
}

.bar{
  /* 占位置 */
}

/*如果要加 DOM 类型，那就写 div.foo.bar */
.foo.bar{
  /* 占位置 */
}
```

## ESLint 和 Pretter

规范一些比较好呐。  
一个是 JS（TS）的检查器，一个是格式化器。  
用 `eslint.config.js` 与 `.pretterc` 进行配置，装了 VSCode 插件就能自动读取到项目根目录的文件

## :class 与 :display

> 参考：[Class 与 Style 绑定](https://cn.vuejs.org/guide/essentials/class-and-style)

`:class` 是 `v-bind:class` 的简写，区别在于能否使用响应式变量。  
最常用的应用是可选控制样式。  
`:class` 与 `class` 是叠加关系，不会覆盖原有的量。
