# 23 — vite.config.ts（展示網站建置設定）

## 兩份 Vite 設定的分工

| 設定檔 | 用途 | 執行指令 |
|--------|------|---------|
| `vite.config.ts` | 展示網站（開發 + 建置）| `pnpm dev` / `pnpm build:docs` |
| `vite.lib.config.ts` | 函式庫打包 | `pnpm build:lib` |

`vite config.ts` 是預設設定，`pnpm dev` 和 `pnpm build:docs` 都使用它。

---

## 完整的 vite.config.ts

```ts
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'
import { defineConfig } from 'vite'
import Markdown from 'unplugin-vue-markdown/vite'
import MarkdownIt from 'markdown-it'
import hljs from 'highlight.js'
import mdConfig from './md.config'

const markdownRenderer = new MarkdownIt()

export default defineConfig({
  build: {
    outDir: 'docs-dist',
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (!id.includes('node_modules')) return

          if (id.includes('element-plus')) return 'vendor-element-plus'
          if (id.includes('highlight.js')) return 'vendor-highlight'
          if (id.includes('/vue/') || id.includes('vue-router')) return 'vendor-vue'
          return 'vendor'
        },
      },
    },
  },
  server: {
    host: '0.0.0.0'
  },
  plugins: [
    vue({
      include: [/\.vue$/, /\.md$/],  // 讓 Vue 外掛也處理 .md 檔案
    }),
    vueJsx(),
    Markdown({
      markdownItOptions: { ... },
      markdownItSetup(md) { mdConfig(md) },
      wrapperClasses: 'markdown-body',
    }),
  ]
})
```

---

## 設定詳解

### outDir：輸出目錄

```ts
build: {
  outDir: 'docs-dist',
```

建置展示網站輸出到 `docs-dist/`，而不是預設的 `dist/`。

這樣 `dist/`（函式庫）和 `docs-dist/`（展示網站）不會衝突。

---

### manualChunks：手動切割 Bundle

```ts
manualChunks(id) {
  if (!id.includes('node_modules')) return  // 自己的程式碼不切割

  if (id.includes('element-plus')) return 'vendor-element-plus'
  if (id.includes('highlight.js')) return 'vendor-highlight'
  if (id.includes('/vue/') || id.includes('vue-router')) return 'vendor-vue'
  return 'vendor'
}
```

這個函數告訴 Rollup 如何把程式碼切割成多個 JS 檔案（chunks）。

**為什麼要切割？**

如果所有東西打包成一個 `bundle.js`，第一次加載需要等整個檔案下載完。

切割後：

```
vendor-vue.js          → Vue 核心（最常用，瀏覽器快取後幾乎不需要重下載）
vendor-element-plus.js → Element Plus（體積最大）
vendor-highlight.js    → highlight.js（只在有程式碼時才用）
vendor.js              → 其他 node_modules
index.js               → 自己的程式碼（最常改動）
```

使用者第一次訪問下載所有 chunk。之後只要你自己的程式碼改變，只有 `index.js` 需要重新下載，`vendor-vue.js` 這些可以直接從快取取用。

---

### server.host

```ts
server: {
  host: '0.0.0.0'
}
```

`0.0.0.0` 讓開發伺服器監聽**所有網路介面**，而不只是 `localhost`。

這樣局域網裡的其他裝置（如手機）可以通過 IP 訪問開發伺服器，方便在手機上測試觸控操作。

---

### Vue 外掛的 include 設定

```ts
vue({
  include: [/\.vue$/, /\.md$/],
})
```

告訴 `@vitejs/plugin-vue` 也要處理 `.md` 檔案（把 `.md` 當成 Vue SFC 處理）。

這是配合 `unplugin-vue-markdown` 的必要設定：
1. `unplugin-vue-markdown` 先把 `.md` 轉換成 Vue SFC 格式
2. `@vitejs/plugin-vue` 再把這個 SFC 編譯成 JS

---

### Markdown 外掛完整設定

```ts
Markdown({
  markdownItOptions: {
    html: true,
    linkify: true,
    typographer: true,
    xhtmlOut: true,
    highlight: (str: string, lang: string) => {
      if (lang && hljs.getLanguage(lang)) {
        try {
          return '<pre class="hljs"><code>' +
            hljs.highlight(str, { language: lang }).value +
            '</code></pre>'
        } catch (__) {}
      }
      return '<pre class="hljs"><code>' + markdownRenderer.utils.escapeHtml(str) + '</code></pre>'
    }
  },
  markdownItSetup(md) {
    mdConfig(md)
  },
  wrapperClasses: 'markdown-body',
}),
```

**`xhtmlOut: true`**：輸出符合 XHTML 標準的 HTML（單標籤自閉合，如 `<br />`）。

**`highlight` 函數**：

```ts
highlight: (str: string, lang: string) => {
  if (lang && hljs.getLanguage(lang)) {
    try {
      return '<pre class="hljs"><code>' +
        hljs.highlight(str, { language: lang }).value +
        '</code></pre>'
    } catch (__) {}
  }
  // fallback：沒有指定語言，直接輸出（HTML 轉義避免 XSS）
  return '<pre class="hljs"><code>' + markdownRenderer.utils.escapeHtml(str) + '</code></pre>'
}
```

`hljs.highlight(str, { language: lang }).value` 把程式碼字串轉換成帶有語法高亮 class 的 HTML。

如果指定的語言不支援，用 `escapeHtml` 安全輸出。

---

## 完整建置流程

```bash
# 開發（兩套同時開發）
pnpm dev                  # 展示網站（使用 vite.config.ts）

# 建置
pnpm build:lib            # 打包函式庫 → dist/
pnpm build:docs           # 打包展示網站 → docs-dist/
pnpm build                # 以上兩者都做
```

建置後的 `docs-dist/` 可以直接部署到任何靜態網站服務（Vercel、GitHub Pages、Netlify）。

---

## vercel.json：Vercel 部署設定

```json
{
  "buildCommand": "pnpm build:docs",
  "outputDirectory": "docs-dist"
}
```

告訴 Vercel：
- 建置指令是 `pnpm build:docs`（不是預設的 `pnpm build`）
- 輸出目錄是 `docs-dist`（不是預設的 `dist`）

---

## 小結

展示網站的建置設定重點：

| 設定 | 效果 |
|------|------|
| `outDir: 'docs-dist'` | 不覆蓋函式庫的 `dist/` |
| `manualChunks` | 依賴分包，提升加載速度 |
| `server.host: '0.0.0.0'` | 手機可訪問開發伺服器 |
| `vue({ include: [/\.md$/] })` | 讓 `.md` 被當成 Vue 元件處理 |
| `Markdown({ highlight })` | Markdown 程式碼塊語法高亮 |

---

## 完整路線回顧

恭喜完成了整個教學！從頭到尾走過：

```
01 環境建置           → 有了空白專案
02 Vue 3 入門         → 理解核心語法
03 lib/ 規劃          → 知道如何組織函式庫
04 型別與常數         → 建立型別地基
05 觀察者模式         → 事件系統基礎
06 EXIF 方向校正      → 解決手機圖片旋轉問題
07 佈局計算           → 圖片尺寸和座標的數學
08 Canvas 操作        → 圖片裁切、邊界計算、動畫
09 縮放與觸控         → 滑鼠 + 觸控統一封裝
10 Canvas 濾鏡        → 像素操作的基本原理
11 JSX 元件           → 輕量元件的另一種寫法
12 樣式               → 多層疊加的視覺設計
13-17 主元件          → 整合所有模組的核心
18 函式庫入口         → Plugin 安裝和導出
19 打包設定           → lib 模式、型別輸出
20-23 展示網站        → Vue Router、Markdown 渲染
```

---

整個 `cropper-next-vue` 函式庫的所有技術細節都在這 24 篇文件裡了。

每一篇都可以獨立查閱，也可以按照順序從頭建構整個專案。
