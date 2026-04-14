# 19 — 打包設定（vite.lib.config.ts + tsconfig.lib.json）

## 為什麼函式庫需要獨立的打包設定？

一般的 Vite 專案打包後得到一個 `index.html` + 一堆 JS/CSS，是給瀏覽器直接執行的。

函式庫的打包不同：
- 不需要 `index.html`
- 需要輸出 **ESM**（`.mjs`）和 **CommonJS**（`.cjs`）兩種格式
- Vue 不能打包進去（使用者自己有 Vue）
- 需要輸出 TypeScript 型別聲明（`.d.ts`）

這就是為什麼要有 `vite.lib.config.ts`（lib 打包）和 `tsconfig.lib.json`（型別輸出），和展示網站的設定分開。

---

## vite.lib.config.ts

```ts
import { resolve } from 'node:path'
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'
import { defineConfig } from 'vite'

export default defineConfig({
  publicDir: false,  // 不複製 public 資料夾（lib 不需要靜態資源）
  build: {
    emptyOutDir: true,  // 每次打包前清空 dist/
    lib: {
      entry: resolve(__dirname, 'lib/index.ts'),  // 入口檔案
      name: 'CropperNextVue',                      // UMD 格式的全域變數名
      formats: ['es', 'cjs'],                      // 輸出格式
      fileName: format => `cropper-next-vue.${format === 'es' ? 'mjs' : 'cjs'}`,
      cssFileName: 'index',                        // CSS 輸出為 index.css
    },
    rollupOptions: {
      external: ['vue'],    // Vue 不打包進去（使用者自己提供）
      output: {
        exports: 'named',   // 使用具名導出
        globals: {
          vue: 'Vue',       // 在 UMD 模式下，vue 對應到全域的 Vue
        },
      },
    },
  },
  plugins: [
    vue(),
    vueJsx(),   // 支援 .tsx 裡的 JSX 語法
  ],
})
```

---

### lib.entry：入口點

```ts
entry: resolve(__dirname, 'lib/index.ts')
```

`resolve(__dirname, ...)` 把相對路徑轉成絕對路徑。

Rollup（Vite 的打包引擎）會從 `lib/index.ts` 開始，找出所有依賴，打包成一個或多個 bundle。

---

### formats：輸出格式

```ts
formats: ['es', 'cjs']
```

**ES Module（`es`）**：現代格式，輸出 `.mjs`。

```js
// .mjs 的格式
export { VueCropper }
export default globalCropper
```

適用於：Vite、Webpack 5、Rollup、現代瀏覽器。

**CommonJS（`cjs`）**：Node.js 傳統格式，輸出 `.cjs`。

```js
// .cjs 的格式
const VueCropper = require('./vue-cropper.vue')
module.exports = { VueCropper }
```

適用於：Webpack 4、Node.js、舊版打包工具。

---

### external: ['vue']：外部依賴

```ts
external: ['vue']
```

告訴 Rollup「vue 這個套件不要打包進去，執行時由外部提供」。

如果不設定，Rollup 會把 Vue 的所有程式碼也打包進 `cropper-next-vue.mjs`，讓打包後的檔案超過 1MB，而且使用者的專案裡會有兩份 Vue，造成錯誤。

---

### cssFileName：CSS 輸出檔名

```ts
cssFileName: 'index'
```

輸出為 `dist/index.css`。

使用者需要單獨引入 CSS：

```ts
import 'cropper-next-vue/style.css'
// 或
import 'cropper-next-vue/dist/index.css'
```

這是函式庫的慣例：CSS 不自動注入（使用者可能有自己的樣式管理方式）。

---

### exports: 'named'

```ts
output: {
  exports: 'named',
}
```

告訴 Rollup 使用**具名導出**的模組格式。

因為 `lib/index.ts` 同時有 `export default` 和 `export { VueCropper }`（具名導出），用 `named` 模式讓兩者都能正常使用。

---

## 執行打包

```bash
pnpm build:lib
# 等同於：
vite build -c vite.lib.config.ts && vue-tsc -p tsconfig.lib.json
```

兩個步驟：

1. **`vite build -c vite.lib.config.ts`**：打包 JS + CSS
2. **`vue-tsc -p tsconfig.lib.json`**：輸出 TypeScript 型別聲明（`.d.ts`）

打包後的 `dist/` 目錄：

```
dist/
├── cropper-next-vue.mjs    ← ES Module 格式
├── cropper-next-vue.cjs    ← CommonJS 格式
├── index.css               ← 樣式
└── types/                  ← TypeScript 型別聲明
    ├── vue-cropper.vue.d.ts
    ├── index.d.ts
    ├── interface.d.ts
    └── ...（每個 lib/ 檔案對應一個 .d.ts）
```

---

## tsconfig.lib.json：型別聲明的 TypeScript 設定

```json
{
  "extends": "./tsconfig.json",  // 繼承主 tsconfig
  "compilerOptions": {
    "declaration": true,         // 輸出 .d.ts 型別聲明
    "emitDeclarationOnly": true, // 只輸出 .d.ts，不輸出 JS（JS 已由 Vite 處理）
    "outDir": "./dist/types",    // .d.ts 輸出位置
    "rootDir": "./lib"           // 只處理 lib/ 目錄
  },
  "include": ["lib/**/*.ts", "lib/**/*.d.ts", "lib/**/*.tsx", "lib/**/*.vue", "shims-vue.d.ts"],
  "exclude": ["src", "docs-dist", "dist"]
}
```

**`extends`**：繼承主 tsconfig 的設定（`strict: true`、`jsxImportSource: vue` 等），避免重複。

**`declaration: true`**：讓 `tsc` 為每個 `.ts`/`.vue` 檔案輸出對應的 `.d.ts` 型別聲明。

**`emitDeclarationOnly: true`**：只輸出 `.d.ts`，不輸出 `.js`。因為 JS 已由 Vite/Rollup 打包完成，這裡只需要型別。

---

## tsconfig.json：主設定

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "preserve",
    "jsxImportSource": "vue",
    "sourceMap": true,
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "types": ["vite/client", "vue/jsx"],
    "lib": ["esnext", "dom"]
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.tsx", "src/**/*.md", "src/**/*.vue", "lib/*", "shims-vue.d.ts"]
}
```

重要選項說明：

| 選項 | 說明 |
|------|------|
| `"strict": true` | 開啟所有嚴格型別檢查（非常建議保持開啟）|
| `"jsx": "preserve"` | JSX 語法保留（不轉換），讓 Vite 的 JSX 外掛處理 |
| `"jsxImportSource": "vue"` | JSX 的自動 import 來源是 Vue（而不是 React）|
| `"resolveJsonModule": true` | 允許 `import foo from './foo.json'` |
| `"moduleResolution": "bundler"` | 適用於 Vite/Rollup 的模組解析策略 |
| `"skipLibCheck": true` | 跳過 node_modules 的型別檢查（加速）|

---

## shims-vue.d.ts：讓 TypeScript 認識 .vue 檔案

```ts
// shims-vue.d.ts（專案根目錄）
declare module '*.vue' {
  import type { DefineComponent } from 'vue'
  const component: DefineComponent<{}, {}, any>
  export default component
}
```

TypeScript 預設不認識 `.vue` 副檔名。這個 shim 告訴 TypeScript：「任何 `.vue` 檔案都是一個 Vue 元件，可以被 import。」

---

## 小結

| 檔案 | 用途 |
|------|------|
| `vite.lib.config.ts` | 打包函式庫：JS（ESM + CJS）+ CSS |
| `tsconfig.lib.json` | 輸出型別聲明（`.d.ts`）|
| `tsconfig.json` | 主 TS 設定，展示網站和 lib 共用 |

執行 `pnpm build:lib` 後：

```
lib/index.ts
    ↓ vite.lib.config.ts
dist/cropper-next-vue.mjs  (.cjs)  index.css
    ↓ tsconfig.lib.json
dist/types/*.d.ts
```

下一篇：[20 — src/main.ts + App.vue + Vue Router（Demo 網站基礎架構）](./20-demo-main.md)
