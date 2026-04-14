# 01 — 環境建置：從零建立空白專案

## 你需要的工具

| 工具 | 說明 | 安裝來源 |
|------|------|---------|
| Node.js 22.x | JavaScript 執行環境 | https://nodejs.org |
| pnpm | 套件管理工具（比 npm 快且省空間）| `npm install -g pnpm` |

確認安裝成功：

```bash
node -v     # 應顯示 v22.x.x
pnpm -v     # 應顯示 9.x.x
```

---

## 為什麼用 pnpm 而不是 npm？

本專案的 `package.json` 指定了 `"packageManager": "pnpm@9.15.9"`，代表這個專案預期用 pnpm 管理。

pnpm 的優點：
- 套件存在硬碟的**共用快取**，多個專案不重複下載
- `node_modules` 結構更嚴格，不會意外引用到沒宣告的套件

---

## 建立專案

### 步驟一：用 Vite 建立 Vue 3 + TypeScript 專案

```bash
pnpm create vite cropper-next-vue --template vue-ts
cd cropper-next-vue
```

這個指令會建立如下結構：

```
cropper-next-vue/
├── public/
├── src/
│   ├── assets/
│   ├── components/
│   ├── App.vue
│   └── main.ts
├── index.html
├── package.json
├── tsconfig.json
└── vite.config.ts
```

> **什麼是 Vite？**  
> Vite 是一個現代前端建置工具。開發時利用瀏覽器原生的 ES module，啟動速度極快。打包時用 Rollup 輸出。

### 步驟二：安裝基本依賴

```bash
pnpm install
```

### 步驟三：確認可以執行

```bash
pnpm dev
```

瀏覽器開啟 `http://localhost:5173`，看到 Vite + Vue 預設畫面即成功。

---

## 調整 package.json

把預設的 `package.json` 改成符合本專案的設定。以下是最終完整版，逐段說明：

```json
{
  "name": "cropper-next-vue",
  "version": "0.1.2",
  "description": "A Vue 3 image cropper focused on rotation, boundary control, and clean library distribution.",
  "packageManager": "pnpm@9.15.9"
```

**`name`**：npm 上的套件名稱，安裝時就是 `npm install cropper-next-vue`。  
**`version`**：版本號，遵循 [語意化版本](https://semver.org/lang/zh-TW/)（主版.次版.修訂）。  
**`packageManager`**：指定 pnpm 版本，讓團隊和 CI 使用一致的工具。

```json
  "main": "./dist/cropper-next-vue.cjs",
  "module": "./dist/cropper-next-vue.mjs",
  "types": "./dist/types/index.d.ts",
  "style": "./dist/index.css",
```

**`main`**：CommonJS 入口（給 Node.js / Webpack 4 用）。  
**`module`**：ES Module 入口（給 Vite / Webpack 5 / Rollup 用）。  
**`types`**：TypeScript 型別宣告檔位置。  
**`style`**：CSS 檔位置（使用者需另行引入）。

```json
  "exports": {
    ".": {
      "types": "./dist/types/index.d.ts",
      "import": "./dist/cropper-next-vue.mjs",
      "require": "./dist/cropper-next-vue.cjs"
    },
    "./style.css": "./dist/index.css",
    "./dist/index.css": "./dist/index.css",
    "./package.json": "./package.json"
  },
```

**`exports`**：Node.js 12+ 的新欄位，精確控制「哪個路徑對應哪個檔案」。  
現代打包工具（Vite、Webpack 5）優先讀 `exports` 而不是 `main`/`module`。

- `import` 條件：用 `import` 語法時走 `.mjs`
- `require` 條件：用 `require()` 時走 `.cjs`
- `"./style.css"` 讓使用者可以 `import 'cropper-next-vue/style.css'`

```json
  "files": ["dist", "README.md", "LICENSE"],
```

**`files`**：發布到 npm 時只包含這些檔案，避免把原始碼、測試、設定檔一起上傳。它在 `npm publish` 或 `npm pack` 時才會生效；這個專案可直接執行 `npm publish`，也可用 `pnpm run release:npm` 包一層版本更新、檢查與發布流程。

```json
  "sideEffects": ["**/*.css"],
```

**`sideEffects`**：告訴打包工具「除了 CSS 之外，這個套件的 JS 沒有副作用」。  
這讓 tree-shaking（搖樹優化）可以安全地移除沒用到的程式碼。這裡的樣式副作用是指 `import './index.css'` 之後會直接影響頁面外觀，所以不能被當成沒用的匯入刪掉。

```json
  "scripts": {
    "dev": "vite",
    "build": "pnpm run build:lib && pnpm run build:docs",
    "build:docs": "vite build",
    "build:lib": "vite build -c vite.lib.config.ts && vue-tsc -p tsconfig.lib.json",
    "typecheck": "vue-tsc --noEmit && ...",
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "check": "pnpm run typecheck && pnpm run test:coverage && pnpm run build:lib",
    "prepublishOnly": "pnpm run check",
    "release:npm": "bash ./scripts/release-npm.sh"
  },
```

重點腳本說明：

| 指令 | 說明 |
|------|------|
| `dev` | 啟動展示網站開發伺服器 |
| `build:lib` | 打包函式庫（Vite 打 JS + vue-tsc 輸出型別） |
| `build:docs` | 打包展示網站 |
| `prepublishOnly` | 執行 `npm publish` 前**自動**跑 check，確保品質 |
| `check` | 型別檢查 + 測試覆蓋率 + 建置，全過才算合格 |

```json
  "engines": {
    "node": "22.x"
  },
```

**`engines`**：限制 Node.js 版本，避免在舊版本執行出錯。

---

## 安裝專案需要的所有依賴

```bash
# 展示網站 + 開發工具
pnpm add -D vite @vitejs/plugin-vue @vitejs/plugin-vue-jsx vue vue-router
pnpm add -D typescript vue-tsc @vue/compiler-sfc

# 測試
pnpm add -D vitest @vue/test-utils jsdom @vitest/coverage-v8

# 展示網站功能
pnpm add -D element-plus highlight.js markdown-it markdown-it-container
pnpm add -D unplugin-vue-markdown sass cheerio

# 型別補充
pnpm add -D @types/node
```

> **DevDependencies vs Dependencies**  
> `-D` 表示加入 `devDependencies`（開發依賴）。 
> `dependencies`：套件會把這個依賴一起帶上。  
> `peerDependencies`：套件不自己帶，要求使用者的專案提供。 


---

## 加入 Vue 的 peerDependency

```bash
pnpm add --save-peer vue@^3.0.0
```

或直接在 `package.json` 加：

```json
"peerDependencies": {
  "vue": "^3.0.0"
}
```

**為何不把 Vue 放在 dependencies？**  
若把 Vue 放進 dependencies，使用者安裝時會得到**兩份 Vue**（他自己一份 + 你這份），造成 Vue 實例衝突。`peerDependencies` 告訴 npm「我需要 Vue，但請用使用者已有的那份」。

---

## 初始化 TypeScript 設定

```bash
pnpm tsc --init
```

本專案有兩份 tsconfig：

| 檔案 | 用途 |
|------|------|
| `tsconfig.json` | 展示網站 + 全域設定 |
| `tsconfig.lib.json` | 函式庫打包專用（只編譯 lib/）|

詳細設定在 [19-build-config.md](./19-build-config.md) 說明。

---

## 建立目錄結構

```bash
mkdir lib
mkdir lib/style
mkdir lib/touch
mkdir lib/filter
mkdir lib/typings
mkdir src
mkdir src/pages
mkdir src/components
mkdir src/composables
mkdir src/assets
mkdir tests
mkdir scripts
```

完成後你的專案根目錄應該長這樣：

```
cropper-next-vue/
├── lib/           ← 函式庫（我們要寫的主要內容）
├── src/           ← 展示網站
├── tests/         ← 測試
├── scripts/       ← 發布腳本
├── public/
├── index.html
├── package.json
├── tsconfig.json
└── vite.config.ts
```

下一篇：[02 — Vue 3 快速入門](./02-vue3-primer.md)
