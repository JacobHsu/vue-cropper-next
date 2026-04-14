# 21 — src/components/ 各元件說明

## 元件概覽

```
src/components/
├── Demo.vue           ← 展示「範例 + 原始碼」的容器
├── SideBar.vue        ← 左側導航選單（桌面 + 手機響應式）
├── LocaleSwitch.vue   ← 中文/英文切換按鈕
├── LangBlock.vue      ← 多語言內容容器（根據語言顯示/隱藏）
├── LangText.vue       ← 多語言文字（內嵌在段落中）
├── DemoImageSwitch.vue ← 切換示範圖片的按鈕組
└── CropExportPanel.vue ← 裁切結果導出面板
```

---

## Demo.vue：範例展示容器

> **Demo.vue 是展示網站（`src/`）的元件，不是函式庫（`lib/`）的一部分。**
> 它只在文件網站上運行，打包函式庫時完全不包含它。

Demo.vue 的職責是把「可以跑的範例」和「可以收起的原始碼」包在一起，讓文件頁面的每個範例都有統一的外觀。

### 完整程式碼（很短）

```vue
<!-- src/components/Demo.vue -->
<template>
  <div class="at-component__container">
    <!-- 實際運行的範例 -->
    <div class="at-component__sample">
      <slot name="demo"></slot>
    </div>

    <!-- 原始碼區塊（可收起）-->
    <div class="at-component__code" v-show="isShow">
      <slot name="sourceCode"></slot>
    </div>

    <!-- 收起/展開按鈕 -->
    <a class="at-component__code-toggle" @click="isShow = !isShow">
      {{ isShow ? '隱藏程式碼' : '顯示程式碼' }}
    </a>
  </div>
</template>

<script setup>
import { ref } from 'vue'
const isShow = ref(true)  // 預設展開原始碼
</script>
```

Script 只有一行有意義的邏輯：`isShow` 控制原始碼區塊的顯示/隱藏，點按鈕時切換。

---

### 兩個具名插槽（Named Slot）

Demo.vue 本身不知道「範例長什麼樣」——它只提供兩個插槽讓外部填入內容：

| 插槽 | 填入什麼 |
|------|---------|
| `#demo` | 實際可互動的 VueCropper 範例 |
| `#sourceCode` | 高亮後的 HTML 原始碼字串 |

---

### 從 `::: demo` 到畫面的完整流程

你在 Markdown 裡寫的是：

```md
::: demo
```vue
<template>
  <VueCropper :img="imgUrl" />
</template>
<script setup>
const imgUrl = 'https://...'
</script>
```
:::
```

這段文字經過四道處理才變成畫面：

```
::: demo 語法塊（寫在 .md 檔案裡）
      ↓
  unplugin-vue-markdown   ← Vite 插件，把整個 .md 檔案編譯成 Vue SFC
  （vite.config.ts 設定）   在這個過程中呼叫 markdown-it 處理 Markdown 語法
      ↓
  md.config.ts            ← markdown-it 的自訂插件（透過 markdownItSetup 注入）
                            專門處理 :::demo 語法塊，把它轉換成 <demo> 標籤：
<demo>
  <template #demo>        ← striptags() 把程式碼裡的 <script> 抽出來執行
    <VueCropper ... />
  </template>
  <template #sourceCode>  ← 原始碼字串（交給 hljs 高亮顯示）
    <pre class="hljs">...</pre>
  </template>
</demo>
      ↓
  Demo.vue                ← 把兩個插槽放進各自容器，加上收起/展開按鈕
      ↓
  瀏覽器畫面：上方是跑起來的範例，下方是可收起的原始碼
```

簡單說：**`unplugin-vue-markdown` 負責把 `.md` 變成 Vue SFC**，`md.config.ts` 是它的一個設定，專門教它「遇到 `:::demo` 要怎麼處理」。兩者是主從關係，不是平行的。

`md.config.ts` 和 `unplugin-vue-markdown` 的細節在 [22 — Markdown 渲染](./22-markdown-pages.md) 說明。

---

## 多語言系統：useLocale + LangBlock + LangText

### composables/useLocale.ts：語言狀態管理

```ts
export type DocLocale = 'zh' | 'en'

const STORAGE_KEY = 'cropper-next-vue-doc-locale'
const localeState = ref<DocLocale>('zh')
```

**`localeState` 宣告在模組頂層**，不在函數內。

這是 Vue 3 Composable 的一個重要技巧：把 `ref` 放在模組頂層，讓所有呼叫 `useLocale()` 的元件**共用同一個狀態**。

相較之下，如果 `localeState` 在 `useLocale()` 函數內宣告，每次呼叫都會建立新的 ref，各元件有各自的狀態，語言切換就不會同步。

```ts
localeState.value = getSavedLocale()  // 從 localStorage 讀取上次的語言設定
```

模組載入時立刻執行，讓頁面一開始就是上次選擇的語言。

```ts
export const useLocale = () => {
  const setLocale = (value: DocLocale) => {
    localeState.value = value
    window.localStorage.setItem(STORAGE_KEY, value)  // 持久化到 localStorage
    document.documentElement.lang = value === 'en' ? 'en' : 'zh-CN'  // 更新 HTML lang 屬性
  }

  return {
    locale: computed(() => localeState.value),
    isZh: computed(() => localeState.value === 'zh'),
    isEn: computed(() => localeState.value === 'en'),
    setLocale,
    toggleLocale,
  }
}
```

**`document.documentElement.lang`**：設定 `<html lang="zh-CN">`，讓螢幕閱讀器和搜尋引擎知道頁面語言。

---

### LocaleSwitch.vue：語言切換按鈕

```html
<section class="locale-switch">
  <button :class="{ active: locale === 'zh' }" @click="setLocale('zh')">中文</button>
  <button :class="{ active: locale === 'en' }" @click="setLocale('en')">EN</button>
</section>
```

設計簡潔：兩個按鈕，點擊後更新 `localeState`，所有使用 `useLocale()` 的元件自動響應更新。

---

### LangBlock.vue：多語言內容容器

用於在 Markdown 裡包裝「只有中文」或「只有英文」的段落：

```html
<!-- 用法（在 Markdown 裡）-->
<LangBlock lang="zh">
這段只在中文模式下顯示。
</LangBlock>
<LangBlock lang="en">
This only shows in English mode.
</LangBlock>
```

實作原理：用 `v-show` 根據當前語言決定顯示或隱藏。

---

### LangText.vue：多語言內嵌文字

```html
<!-- 用法（在 Markdown 裡）-->
<p>
  點擊 <LangText zh="確定" en="Confirm" /> 按鈕
</p>
```

比 `LangBlock` 更輕量，適合在句子中間插入多語言文字。

---

## SideBar.vue：導航選單

```ts
const { isEn } = useLocale()

const navGroups = computed(() => [
  {
    title: isEn.value ? 'Docs' : '文件',
    items: [
      { name: isEn.value ? 'Guide' : '快速開始', path: '/guide' },
      { name: isEn.value ? 'Props' : '參數', path: '/props' },
      // ...
    ]
  },
  // ...
])
```

導航選單的文字根據語言動態改變。這樣語言切換後，選單也會立刻更新，不需要重新整理頁面。

**`router-link`**：Vue Router 提供的元件，會渲染成 `<a>` 標籤，點擊時切換路由而不刷新頁面。

```html
<router-link class="item-link" :to="item.path" @click="mobile && emit('close')">
  {{ item.name }}
</router-link>
```

手機版的 `@click="mobile && emit('close')"` 是一個簡潔的條件執行：

- 桌面版（`mobile = false`）：`false && emit(...)` → 短路，不執行 `emit`
- 手機版（`mobile = true`）：`true && emit('close')` → 執行 `emit('close')`，關閉側邊欄

---

## DemoImageSwitch.vue：示範圖片切換

在 Demo 頁面頂部提供幾張預設圖片，讓使用者快速切換展示效果：

```html
<DemoImageSwitch v-model="currentImg" />
```

元件內部用 `v-model` 雙向綁定：使用者點選不同圖片，父元件的 `currentImg` 自動更新。

---

## CropExportPanel.vue：裁切導出面板

在 Demo 頁面提供導出按鈕和預覽區域：

- 點擊「Get Data（base64）」→ 呼叫 `cropperRef.getCropData()` → 顯示 base64 結果
- 點擊「Get Blob」→ 呼叫 `cropperRef.getCropBlob()` → 顯示 Blob URL

---

## 元件互動關係

```
App.vue
 ├── SideBar（使用 useLocale 讀取語言）
 │    └── LocaleSwitch（呼叫 useLocale 改語言）
 └── router-view
      └── Markdown 頁面（通過全域 components 使用）
           ├── <demo>（Demo.vue）
           │    ├── slot#demo：實際展示的 VueCropper
           │    └── slot#sourceCode：高亮原始碼
           ├── <LangBlock>：多語言段落
           ├── <LangText>：多語言文字
           ├── <DemoImageSwitch>：圖片切換
           └── <CropExportPanel>：導出功能
```

---

## 小結

這些元件的共同設計原則：

1. **職責單一**：Demo 只管展示，SideBar 只管導航，LocaleSwitch 只管切換語言
2. **共享狀態**：`useLocale` 的模組頂層 ref 確保所有元件看到同一個語言狀態
3. **全域可用**：在 `main.ts` 全域註冊，讓 Markdown 頁面可以直接使用這些元件

下一篇：[22 — src/pages/ + md.config.ts（Markdown 渲染）](./22-markdown-pages.md)
