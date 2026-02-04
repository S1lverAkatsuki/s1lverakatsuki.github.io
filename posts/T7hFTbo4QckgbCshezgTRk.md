# Vue 中用 Vite 设置转义 @ 路径

*谁都不想写一大堆相对路径*
配置完成后是用 `@` 代表 `src` 文件夹的路径。
无论是 Tauri 还是纯粹的前端项目都可用的方法。

## 更改 `vite.config.ts`

```ts
import path from "path";

export default defineConfig(() => ({
  // ...

  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src')
    }
  },
}));
```

`path` 是 NodeJS 自带的包，如果出现 TS 的类型报错（找不到类型提示）就把类型包装一下：

```bash
pnpm add @types/node --save-dev
```

*我这里是 pnpm，用你自己的包管理器装*

## 更改 `tsconfig.json`

```json
{
  "compilerOptions": {
    // ...

    "paths": {
      "@/*": ["./src/*"]
    },

    // ...
  },
}
```
