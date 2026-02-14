# simply-writer 开发笔记

[仓库地址](https://github.com/S1lverAkatsuki/simply-writer)

## 前端部分

### 隐藏元素

让这个元素直接在渲染中消失，而且**不会影响页面布局**：

```css
.u-cannt-see-me {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  border: 0;
  clip: rect(0 0 0 0);
  overflow: hidden;
}
```

### 触发原生元素的事件

这里有个 HTML 的 DOM：

```html
<input type="file" ref="fileSelectorRef">
```

它是用来在前端上传文件为数不多的手段，因为不能用任何纯 JS 的方式来上传文件。
怎么在 JS 中调用它的方法呢？直接在引用上调用 `click` 方法就好了：

```ts
const triggerFileSelect = () => fileSelectorRef.value?.click();
```

### 多状态类型

有一个场景：有学生和老师两个用户，学生需要记录成绩，老师需要记录课程。
可以把这两个需求存在一种对象里面：

```ts
type User = { type: "student"; score: number } | { type: "teacher"; class: string };
```

在用的时候，只需要先对 `type` 进行判定，后续 TS 会自己缩窄类型到对应的字段：

```ts
const user1: User = ...;
if (user1.type === "student") {
  // 可以访问 user1.score 字段
  // ...
} else {
  // 可以访问 user1.class 字段
  // ...
}
```

### `transform: scale` 与真实的 DOM 大小

`transform: scale(2)` 只会让 DOM 看起来更大，但是在文档流里面的大小还是一定的。
最好同步修改 `width` 和 `height` 实现真正的缩放。

### 字符串类型打印的解决方案

现在有一个性别类型，它可以封装几个确定的值作为选择，避开了 `boolean` 的意义不明：

```ts
type Sex = "Man" | "Women";
```

但是如果我要把它打出来呢？类型是不能当值用的，对吧。
只需要用值生成类型就行了，打印就直接打印原本的值：

```ts
const SEX = ["Man", "Women"] as const;
type Sex = (typeof SEX)[number];
```

### 前端保存文本文件

我看了一圈好像都没的说的，这里让我总结一下吧。
简单来说就是靠点击 `<a download="路径">`来下载含有文本文件 `Blob` 的 URL。

```ts
const save = async (payload: string) => {
  cosnt blob = new Blob([payload], { type: "text/plain;charset=utf-8" );
  let downloadUrl = URL.createObjectURL(blob);
  // dowloadUrl 靠 Vue 响应式变量之类的塞进 a 标签内
  // 然后靠前文提到的触发 a 标签的方式
};
```

这里没做任何错误处理和预防，最好对 `downloadUrl` 进行检查。
如果存在可以靠 `URL.revokeObjectURL(downloadUrl)` 和手动置 `downloadUrl = null` 释放 `ObjectURL`。

## 后端部分

见《[非常简单地上手 Axum 后端](https://s1lverakatsuki.github.io/#/article?uuid=N9QQgws38Udf4zGVDc6RPd)》，由此总结出的后端（包括前端向后端的请求）的理解都在里面了。
