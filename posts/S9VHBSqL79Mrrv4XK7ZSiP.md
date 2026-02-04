# LLM-Prompt-Manager 开发笔记

[仓库地址](https://github.com/S1lverAkatsuki/llm-prompt-manager)

## 前端

### Vue 的加载时间

在开启应用的时候，总会有个白屏的时候，过了大概一两秒才变成写好的 UI。
这其实是前端**挂载 Vue 组件的过程**，显示的是 `index.html` 里的东西。
才不是什么卡 IO 呢！

### 响应式丢失 & Vue 修改响应式变量推荐规则

在 v-for 里面，一般会解构出一个数组中的值进行操作：

```ts
<script setup lang="ts">
type Item = {
  data: number;
}

const list = ref<Item[]>([{data: 1}, {data: 2}, {data: 3}]);

const updateItem = (item: Item) => item = {data: 2};
</script>

<div v-for="item in list" @click="updateItem(item)">
    {{ item.data }}
</div>
```

`updateItem` 直接对局部变量 `item` **重新赋值**，这仅仅是让局部变量 item 指向了新地址（对，这是个引用），也没有改变原来的对象。

这种必须**替换整个对象**的操作会丢失响应性。

但如果像是下面这样：

```ts
// ...
const updateItem = (item: Item) => item.data = 2;
```

对其具有的**属性**进行修改，这个就能被 Proxy 捕获到正常响应更新。

如果非得要整个替换属性，那还是**传索引改原对象**吧：

```ts
<script setup lang="ts">
// ...
const updateItem = (idx: number) => list[idx] = 2;
</script>

<template>
  <div v-for="(item, index) in list" @click="updateItem(index)">
    {{ item.data }}
  </div>
</template>
```

### v-if 优化 null 检查

经常遇到需要当某个未初始化的响应式变量的值为非 null 时候才要进行的逻辑。

```ts
<script setup lang="ts">
const foo = ref<number | null>(null);
</script>

<div v-if="foo" >
  <!-- 这里的逻辑都会认为 foo 非空 -->
</div>
```

如果有好几个，一个个写 `v-if` 会增加渲染负担，用 `<template>` 包起来：

```ts
<template v-if="foo">
  <div @click="doSomething1(foo)" />
  <div @click="doSomething2(foo)" />
</template>
```

用 `v-if` 排除非空时，记得在实际处理函数中加上 `await nextTick()` 等待 DOM 创建。

### 异步逻辑封装为同步 Hook

这里有个人畜无害的异步函数：

```ts
const foo = async (num: number): Promise<number> => {
  // 这里有一些异步操作
  return number + 42;
};
```

一般来说，我们都是这样用它的：

```ts
const data = ref<number | null>(null);

onMounted(async () => {
  const result = await foo(123);

  // 等待 foo 跑完，没跑完就卡在这里

  data.value = result;
});
```

可以靠一个神秘函数把它转成类似于**同步**的写法：

```ts
// 这是个同步函数
export const useSyncFoo = (num: number) {
  // 1. 用 data 这个响应式变量占位
  const data = ref<number | null>(null);
  const isFinished = ref<boolean>(false);

  // 2. 启动异步逻辑，但不对外阻塞
  foo(number).then((res) => {
    // 3. 异步完成时，通过响应式系统自动触发 UI 更新
    data.value = res;
    isFinished.value = true;
  });

  // 4. 同步返回响应式变量，哪怕在 foo 没执行完前都是空的
  return { data, isFinished };
}
```

在外面就可以写：

```ts
const { data, isFinished } = useSyncFoo(123);

// 后面的逻辑不会被 await 打断
```

`useSyncFoo` 把等待 `foo` 返回值的过程转换成了 `data` 值的改变。也就是过程式转换成...声明式！ *写到这里的时候去查资料了*

### Tailwind CSS 颜色百分比

`/` 是不透明度修饰符。

```html
<div class="bg-base-content/50"></div>

<div class="text-blue-500/25"></div>
```

即为把 `bg-base-content` 这个类对应的颜色透明度减到 50%

自定义透明度参照下面的写：

```html
<div class="text-base-100/[.33]"></div>
```

### Tailwind CSS 插入 CSS 属性条文

> 参考：[Adding custom styles](https://tailwindcss.com/docs/adding-custom-styles)

总有不能被写成 Tailwind 类的 CSS 属性，比如说控制是否给滚动条留白的 `scrollbar-gutter`。
打个中括号写出来就好了：

```html
<div class="[scrollbar-gutter:stable]"></div>

<div class="top-[117px] bg-[#bada55]"></div>
```

### 用引用给对象赋值

如果有个对象（比如说数组和 `Object`）是引用，把它赋值给另一个对象，就会带来关联：

```ts
const openEditor = async (item?: Prompt) => {
  if (item) {
    // tags 是引用，必须复制才能和原来的脱钩
    editingPrompt.value = {
      title: item.title,
      tip: item.tip,
      content: item.content,
      tags: [...item.tags]
    };

  // ...
};
```

这里最初是直接赋值，由于 `tags` 类型是**数组**，不进行浅拷贝会在修改 `editingPrompt` 时候反过来改原来的变量。

现在想来对象解构会更好：

> 参考：[解构](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operatorust/Destructuring)

```ts
editingPrompt.value = {
  ...item,  // 创建对象副本，其他的都是按值传递的基本类型
  tags: [...item.tags]  // 引用类型单独创建副本
};
```

### 「一段时间后执行」的常见实现

JS 提供了 `setTimeout` 这种会在一定时间后运行回调的方法，看了文档后就很容易写出来这种东西：

```ts
setTimeout(() => console.log("两秒后打印"), 2000);
```

如果在事件 Hook 当中，就会随着事件触发而创建多个定时器，它们只会各自执行自己的约定：一段时间后执行回调。
这在处理诸如按钮按下两秒后变色的逻辑会出现被多个事件反复修改导致表现效果很奇怪。
用一个响应式变量装起来，检查是不是有东西在里面再分别处理：

```ts
const timeout = ref<NodeJS.Timeout | null>(null);

const foo = () => {
  // 当前有定时器在跑
  if (timeout.value) {
    clearTimeout(timeout.value);
  }

  // 重新刷新计时时间
  timeout.value = setTimeout(() => {
    console.log("两秒后打印");
    timeout.value = null;  // 回调运行完了以后不会自动置空
  }, 2000);
}
```

记得卸载组件也要删：

```ts
onUnmounted(() => {
  if (timeout.value) clearTimeout(timeout.value);
});
```

### Daisy UI 使用 JS/TS 切换主题

只要学会按一下 <kbd>F12</kbd>，就能看到在深浅色切换的时候每个元素都改了个 CSS 属性 `data-theme`，依葫芦画瓢即可：

```ts
const switchTheme = (toDarkMode: boolean) => {
  const theme = value ? "dark" : "light";
  document.documentElement.setAttribute("data-theme", theme);
}
```

### Tauri 提供的资源管理器 API

> 参考：[@tauri-apps/plugin-opener](https://v2.tauri.org.cn/reference/javascript/opener/#revealitemindir)

### Tauri 提供的剪切板管理 API

> 参考：[剪贴板](https://v2.tauri.org.cn/plugin/clipboard/)

## 后端

### Tauri 基本架构与「托管后端管理类」的实现

任何一个拿到 Tauri 的人都知道写出这种命令作为后端和前端的接口。

```rust
#[tauri::command]
fn greeting() -> String {
    return String::from("Hello!");
}
```

但如果要涉及到一般情况下，用一个抽象的**全局管理类**来承接接口管理功能呢：

```rust
struct Greeter {
    greet_say: String
}

impl Greeter {
    fn greeting(&self) -> String {
        return self.greet_say.clone();
    }
}
```

这时候可以用 Tauri 提供的状态管理类：

```rust
struct AppState {
    greater: Mutex<Greeter>,
}
```

这里是使用 `Mutex` 上锁防止更改抢占，Tauri 命令是多线程的。
涉及手动跨线程（诸如自己开一个专门的 IO 进程）那还得套一层变成 `Arc<Mutex<T>>`，现在不需要是 Tauri 内部实现了类似 `Arc` 的引用计数。
`RwLock` 读写锁可以在**读多写少**的场景替换 `Mutex` 互斥锁。
如果 `greeter` 是只读的 *显然不可能吧*，那到可以不用加壳。

然后把它挂载到 Tauri 初始化过程当中：

```rust
fn main() {
    tauri::Builder::default()
        // 将全局管理类注入到 Tauri 运行环境中
        .manage(AppState {
            greeter: Mutex::new(Greeter {
                greet_say: "Hello!".into(),
            }),
        })
        .invoke_handler(tauri::generate_handler![greeting])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

命令也得改，有状态就得抢过来占有，解出原始对象后在干原来的操作。

```rust
#[tauri::command]
fn greeting(state: tauri::State<'_, AppState>) -> Result<String, String> {
    let greeter = state.greeter.lock().map_err(|_| "Failed to lock state")?;
    
    Ok(greeter.greeting())
}
```

就这样，这个模板可以涵盖目前我遇到的所有后端管理。

### if let 范式简化错误处理

```rust
const result = Some(42);

if let Some(value) = result {
    println!("得到内容：{}", value);
} else {
    println!("它是空的");
}
```

如果是只关注一种可能（诸如这里只要管里面有东西的情况），就用 if let，而不是 match。
`unwrap_or` 也可以，但是里面闭包写太长不好，if let 更像是顺序逻辑。

### Tauri 提供的系统存储路径 API

> 参考：
>
> - JS 侧：[@tauri-apps/plugin-fs](https://v2.tauri.app/zh-cn/reference/javascript/fs/)
> - Rust 侧：[PathResolve](https://docs.rs/tauri/2.10.1/tauri/path/struct.PathResolver.html)

Rust 侧用例：

```rust
// app 类型为 tauri::App

// A
let doc_dir = app.path().document_dir().expect("获取文档目录失败");
let data_path = doc_dir.join("my_data.json");

// B
let data_path = app.path().resolve("my_data.json", BaseDirectory::Document);
```

两种调用效果一致，都是返回 `C:\Users\<用户名>\Documents\my_data.json` （Windows）
第一种调用先拿到的是文档文件夹的位置，然后再拼上自定义的配置文件位置。
第二种就是 `app.path()` 拿到 `PathResolve` 对象，在调用链上把文档文件夹目录 `BaseDirectory::Document` （这个枚举最好是由 `PathResolve` 对象访问，单独访问枚举值会有一屁股问题）和配置文件名称拼一起。
