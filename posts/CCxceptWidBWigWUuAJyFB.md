# 如何开一个前端项目

包管理 pnpm，框架 Vue，语言 TS，样式表 Tailwind CSS，检查工具 oxc 两件套（lint，fmt）。
这是我的一个常见配置，从创项目开始，需要自备 pnpm 包管理器。
里面还有关于 VSC 配置 Tailwind CSS 补全的选项。

1. 你要创建项目的目录下打开终端内运行：

    ```bash
    pnpm create vite@latest <项目名称> -- --template vue-ts
    ```

    项目名称推荐用短横线分隔小写字母的规则，如 `my-fxxking-project`。

2. 终端进到创建出的目录 `<项目名称>` 内。
3. 安装 Tailwind CSS： `pnpm i -D tailwindcss @tailwindcss/vite`。
4. 安装 oxc 两件套：`pnpm i -D oxlint oxfmt`。
5. 修改 `vite.config.js` 中关于 Tailwind CSS 的配置，同时把 `@` 转义加上：

    ```js
    import { defineConfig } from "vite";
    import vue from "@vitejs/plugin-vue";
    import tailwindcss from "@tailwindcss/vite";
    import path from "path";

    const __dirname = import.meta.dirname;

    export default defineConfig({
      plugins: [ vue(), tailwindcss()],
      resolve: {
        alias: {
        "@": path.resolve(__dirname, "./src"),
        },
      },
    })
    ```

6. 删掉 `src/style.css` 下模板带有的默认样式，导入 Tailwind CSS，让窗口铺满视口。

    ```css
    @import "tailwindcss";

    html,
    body {
      width: 100%;
      height: 100%;
    }
    ```

7. 修改 `package.json` 为其添加 lint 与 fmt 命令：

    ```json
    "scripts": {
      "dev": "vite",
      "build": "vue-tsc -b && vite build",
      "preview": "vite preview",
      "lint": "oxlint",
      "lint:fix": "oxlint --fix",
      "fmt": "oxfmt",
      "fmt:check": "oxfmt --check"
    },
    ```

8. 在项目根目录的 `.vscode` 配置文件夹下创建 `setting.json`，补全如下内容：

    ```json
    {
      "files.associations": {
        "*.css": "tailwindcss"
      },
      "editor.quickSuggestions": {
        "strings": "on"
      }
    }
    ```

    为了让 Tailwind CSS 的补全可以正确生效。

9. `tsconfig.app.json` 下添加 `@` 替换：

    ```json
    {
      "compilerOptions": {
        "paths": {
            "@/*": ["./src/*"]
        },
      }
    }
    ```

10. 根目录下创建 `.oxfmtrc.json`，下面是我常用的格式化限定：

    ```json
    {
      "semi": true,
      "singleQuote": false,
      "trailingComma": "es5",
      "tabWidth": 2,
      "useTabs": false,
      "printWidth": 80,
      "bracketSpacing": true,
      "arrowParens": "avoid",
      "singleAttributePerLine": true
    }
    ```

    含义对应如下：

    - `semi: true`
        表示语句末尾保留分号。
    - `singleQuote: false`
        表示字符串默认使用双引号，不使用单引号。
    - `trailingComma: "es5"`
        表示在 ES5 支持的多行结构中保留尾随逗号，比如对象、数组等；函数参数通常不加。
    - `tabWidth: 2`
        表示缩进宽度按 2 个空格计算。
    - `useTabs: false`
        表示缩进使用空格，不使用 Tab。
    - `printWidth: 80`
        表示一行最大长度倾向控制在 80 个字符内，超出时会自动换行。
    - `bracketSpacing: true`
        表示对象字面量的大括号内保留空格，例如 `{ foo: bar }`。
    - `arrowParens: "avoid"`
        表示箭头函数只有一个参数时尽量省略括号，例如 `x => x + 2`，而不是 `(x) => x + 2`。
    - `singleAttributePerLine: true`
        表示在 HTML / JSX / Vue 这类标签中，每个属性单独占一行。

11. 删掉 `src/App.vue` 里没必要的东西，删掉 `src/components/HelloWorld.vue` 我们不需要模板了。
12. 给 `src/App.vue` 里 `<template>` 下第一个主容器添加类 `w-screen`，`h-screen`，`box-border`。
13. 开始你的表演吧。
