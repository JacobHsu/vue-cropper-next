# 20 — src/main.ts + App.vue + Vue Router（Demo 網站基礎架構）

## Demo 網站的角色

`src/` 是一個完整的 Vue 3 單頁應用（SPA），用來展示 `lib/` 函式庫的所有功能和說明文件。

它和 `lib/` 完全獨立：

```
lib/  ← 函式庫（pnpm build:lib 打包）
src/  ← 展示網站（pnpm build:docs 打包）
```

展示網站直接從 `lib/` import，模擬使用者安裝後的使用體驗：

```ts
// src/main.ts
import VueCropper from '../lib'   // 開發時用相對路徑
// import VueCropper from 'cropper-next-vue'  // 發布後換成這行
```

---

## src/main.ts：應用入口

```ts
import { createApp } from 'vue'
import App from './App.vue'

// 全域元件（在所有 Markdown 頁面裡都可以直接使用）
import Demo from './components/Demo.vue'
import LangBlock from './components/LangBlock.vue'
import LangText from './components/LangText.vue'
import DemoImageSwitch from './components/DemoImageSwitch.vue'
import CropExportPanel from './components/CropExportPanel.vue'

import { createRouter, createWebHashHistory } from 'vue-router'
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'

import VueCropper from '../lib'
```

**為什麼要全域註冊 `Demo`、`LangBlock` 等元件？**

Markdown 頁面（`src/pages/*.md`）會被 `unplugin-vue-markdown` 編譯成 Vue 元件。Markdown 裡可以直接使用 Vue 元件語法，但沒辦法在 Markdown 裡做 `import`。

全域註冊後，所有 Markdown 頁面都能直接使用 `<demo>`、`<LangBlock>` 等標籤。

---

### Vue Router 設定

```ts
const router = createRouter({
  history: createWebHashHistory(),   // hash 模式：URL 用 # 分隔
  routes: [
    { name: 'Home', path: '/', component: () => import('./pages/Home.vue') },
    { name: 'Guide', path: '/guide', component: () => import('./pages/Guide.md') },
    { name: 'Props', path: '/props', component: () => import('./pages/Props.md') },
    // ...共 14 個路由
  ],
})
```

**為什麼用 Hash 模式（`createWebHashHistory`）而不是 HTML5 模式（`createWebHistory`）？**

HTML5 模式（`/guide`）：需要伺服器配合，把所有路徑都回傳 `index.html`。  
Hash 模式（`/#/guide`）：完全在瀏覽器端處理，不需要伺服器配合，適合純靜態部署（如 GitHub Pages、Vercel 的靜態模式）。

**動態 import（`() => import('./pages/Guide.md')`）**：

路由元件用動態 import，這樣每個頁面在**使用者點選時**才載入，而不是一開始就全部載入。

這叫做**路由分割**（Route-based Code Splitting）：把應用拆成多個小 chunk，按需載入，加速首頁載入速度。

---

### 應用初始化

```ts
const app = createApp(App)
app.use(ElementPlus)       // UI 元件庫
app.use(VueCropper)        // 全域安裝裁剪元件
app.use(router)            // 路由

// 全域元件
app.component('demo', Demo)
app.component('LangBlock', LangBlock)
app.component('LangText', LangText)
app.component('DemoImageSwitch', DemoImageSwitch)
app.component('CropExportPanel', CropExportPanel)

app.mount('#app')
```

**執行順序**：`createApp` → `use(外掛)` → `component(全域元件)` → `mount`

`mount('#app')` 把 Vue 應用掛載到 `index.html` 裡的 `<div id="app">` 上。

---

## src/App.vue：根元件

### 功能概述

`App.vue` 是整個應用的框架：
- 左側固定的 `SideBar`（導航選單）
- 右側的 `<router-view>`（路由頁面）
- 手機版的響應式佈局（漢堡選單 + 全螢幕側邊欄）

---

### 響應式偵測

```ts
const isMobile = ref(false)
const sidebarOpen = ref(false)

let mql: MediaQueryList | null = null

const onMediaChange = () => {
  isMobile.value = !!mql?.matches        // 視窗寬度 <= 767px 時，isMobile = true
  if (!isMobile.value) {
    sidebarOpen.value = false             // 切換到桌面時，關閉側邊欄
  }
}

onMounted(() => {
  mql = window.matchMedia('(max-width: 767px)')  // 監聽媒體查詢
  onMediaChange()                                 // 立刻執行一次

  // 舊版 Safari 用 addListener
  if (mql.addEventListener) {
    mql.addEventListener('change', onMediaChange)
  } else {
    mql.addListener(onMediaChange)
  }
})
```

**`window.matchMedia`**：JavaScript 的媒體查詢 API，類似 CSS 的 `@media (max-width: 767px)`，但在 JavaScript 裡可以監聽媒體條件的變化。

**為什麼要同時處理 `addEventListener` 和 `addListener`？**

`addEventListener` 是新 API，`addListener` 是舊 API（已廢棄）。舊版 Safari 和 IE 只支援 `addListener`，需要向下相容。

---

### 路由切換關閉側邊欄

```ts
const route = useRoute()

watch(
  () => route.fullPath,       // 監聽完整 URL 路徑
  () => {
    if (isMobile.value) {
      sidebarOpen.value = false  // 手機版切換頁面後，關閉側邊欄
    }
  }
)
```

`useRoute()` 是 Vue Router 提供的 Composable，回傳當前路由物件。

`route.fullPath` 包含 path + query + hash（`/guide?lang=en#section1`）。

---

### Template 結構

```html
<section
  id="app"
  class="wrapper"
  :class="{
    'is-mobile': isMobile,
    'sidebar-open': isMobile && sidebarOpen,
  }"
>
  <!-- 手機版頂部導覽列 -->
  <header v-if="isMobile" class="top-bar">
    <button @click="sidebarOpen = true">≡</button>
    <span>cropper-next-vue</span>
  </header>

  <!-- 側邊欄 -->
  <SideBar :mobile="isMobile" :open="sidebarOpen" @close="sidebarOpen = false" />

  <!-- 點擊背景關閉側邊欄（手機版）-->
  <div v-if="isMobile && sidebarOpen" class="backdrop" @click="sidebarOpen = false" />

  <!-- 主內容區：路由頁面 -->
  <main class="container">
    <router-view></router-view>
  </main>
</section>
```

**`:class` 物件語法**：

```html
:class="{ 'is-mobile': isMobile }"
```

當 `isMobile` 為 `true` 時，加上 `is-mobile` class；為 `false` 時移除。

---

### 樣式設計重點

```scss
.wrapper {
  display: flex;          // 側邊欄 + 主內容水平排列
  height: 100vh;
  height: 100dvh;         // dvh = 動態視口高度（考慮手機瀏覽器的導覽列）
  overflow: hidden;
}

.container {
  flex: 1;                // 佔滿剩餘寬度
  overflow: auto;         // 內容太長時可以滾動
}
```

**`100dvh` vs `100vh`**：

在手機瀏覽器上，`100vh` 有時包含瀏覽器的工具列高度，導致內容被遮擋。

`100dvh`（Dynamic Viewport Height）會隨工具列的顯示/隱藏自動調整，更準確。

先寫 `100vh` 作為 fallback（舊瀏覽器不支援 `dvh`），再寫 `100dvh` 覆蓋（支援的話用這個）。

---

### 手機版響應式

```scss
.wrapper.is-mobile {
  flex-direction: column;  // 垂直排列（頂部導覽列 + 主內容）
}

.wrapper.is-mobile .top-bar {
  display: flex;           // 手機版才顯示頂部導覽列
  height: 52px;
  padding-top: env(safe-area-inset-top);  // iPhone 瀏海補偏
}

.wrapper.is-mobile .backdrop {
  display: block;          // 半透明背景
  position: fixed;
  inset: 0;                // top:0, right:0, bottom:0, left:0 的簡寫
  background: rgba(0, 0, 0, 0.35);
  z-index: 90;
}
```

**`env(safe-area-inset-top)`**：iPhone X+ 的瀏海和圓角邊距，`env()` 讓 CSS 可以讀取這個值。

---

## 小結

Demo 網站的入口結構：

```
main.ts
  ├── createApp(App)
  ├── app.use(ElementPlus)    ← UI 元件庫
  ├── app.use(VueCropper)     ← 安裝函式庫（全域可用 <VueCropper>）
  ├── app.use(router)         ← 路由
  ├── app.component(...)      ← Markdown 頁面用的全域元件
  └── app.mount('#app')
```

App.vue 負責：
- 整體佈局（側邊欄 + 內容區）
- 響應式偵測（桌面/手機）
- 手機版的側邊欄開關

下一篇：[21 — src/components/ 各元件說明](./21-demo-components.md)
