# 11 — lib/loading.tsx（JSX 元件寫法）

## 為什麼不用 .vue 而用 .tsx？

`.vue` 檔案的三段式結構（`<template>`、`<script>`、`<style>`）非常適合複雜的元件。

但對於**非常簡單的元件**，這個結構顯得繁瑣。

Loading 元件只做一件事：「顯示時，渲染一個旋轉的 SVG；隱藏時，什麼都不渲染」。

用 JSX（`.tsx`）可以把邏輯和結構都寫在同一個函數裡，更精簡：

```
.vue 方式：需要 <template>、<script>、分開來寫
.tsx 方式：一個 render 函數就搞定
```

---

## JSX 是什麼？

JSX 是在 JavaScript 裡寫 HTML 的語法，由 React 發明，Vue 3 也支援。

**標準 HTML 是字串**，不容易和 JavaScript 邏輯混合：

```js
// 字串拼接：難以閱讀，沒有語法高亮
const html = '<div class="foo"><p>' + title + '</p></div>'
```

**JSX 是函數呼叫**，可以直接內嵌 JS 表達式：

```jsx
// JSX：可讀性更高，TypeScript 有型別檢查
const element = (
  <div class="foo">
    <p>{title}</p>
  </div>
)
```

JSX 的 `<div>` 最終會被編譯成 `h('div', { class: 'foo' }, ...)` 這樣的 Vue `h()` 函數呼叫。

---

## 建立 lib/loading.tsx

### defineComponent：定義選項式元件

```ts
import { defineComponent } from 'vue'

const cropperLoading = defineComponent({
  name: 'cropper-loading',

  props: {
    isVisible: {
      type: Boolean,
      default: false
    }
  },
```

**為什麼用 `defineComponent` 而不是 `<script setup>`？**

`<script setup>` 是 `.vue` 檔案的語法。在 `.tsx` 檔案裡必須用 `defineComponent` 或函數式元件的寫法。

`defineComponent` 的好處：可以讓 TypeScript 知道這是一個 Vue 元件，提供型別補全。

---

### render 函數

```ts
  render() {
    const { $slots }: any = this
    return (
      this.isVisible ?
      <section class="cropper-loading">
        { $slots.default ? $slots.default() :
          <p class="cropper-loading-spin">
            <i>
              <svg ...>
                <path d="..." />
              </svg>
            </i>
            <span />
          </p>
        }
      </section>
      : ''
    )
  }
```

**整個 render 函數的邏輯**：

```
isVisible 為 false → 回傳空字串（什麼都不顯示）
isVisible 為 true  → 顯示 section.cropper-loading
  有插槽 → 顯示使用者自訂的 loading 內容
  沒插槽 → 顯示預設的旋轉 SVG
```

---

### 插槽（Slot）在 JSX 裡的寫法

在 `<template>` 語法裡，插槽是 `<slot name="loading" />`。

在 JSX 裡，透過 `$slots` 物件取得插槽內容，`$slots.default()` 呼叫預設插槽的渲染函數：

```jsx
{ $slots.default ? $slots.default() : <p>預設內容</p> }
```

**在 vue-cropper.vue 裡使用**：

```html
<!-- 使用預設 loading（旋轉 SVG）-->
<cropperLoading :is-visible="imgLoading" />

<!-- 使用自訂 loading -->
<cropperLoading :is-visible="imgLoading">
  <template #default>
    <MyCustomLoadingSpinner />
  </template>
</cropperLoading>
```

---

### SVG 說明

```jsx
<svg
  viewBox="0 0 1024 1024"
  focusable="false"
  data-icon="loading"
  width="1.5em"
  height="1.5em"
  fill="currentColor"
  aria-hidden="true"
>
  <path d="M988 548c-19.9 0-36-16.1-36-36 0-59.4-11.6-117..." />
</svg>
```

**`fill="currentColor"`**：SVG 圖示的填充色繼承父元素的 CSS `color`。這樣使用者只需改 `color` 就能更改 loading 顏色，不需要修改 SVG。

**`aria-hidden="true"`**：告訴螢幕閱讀器忽略這個裝飾性圖示，提升無障礙性（Accessibility）。

**旋轉動畫**：這個路徑是一個「有缺口的圓形」（像 C 形），通過 CSS 的 `animation: spin` 讓它不停旋轉，就是經典的 loading 效果。

---

## 完整的 lib/loading.tsx

```tsx
import { defineComponent } from 'vue'

const cropperLoading = defineComponent({
  name: 'cropper-loading',

  props: {
    isVisible: {
      type: Boolean,
      default: false
    }
  },

  render () {
    const { $slots }: any = this;
    return (
      this.isVisible ? 
      <section class="cropper-loading">
        { $slots.default ? $slots.default() : 
         <p class="cropper-loading-spin">
            <i>
              <svg
                viewBox="0 0 1024 1024"
                focusable="false"
                data-icon="loading"
                width="1.5em"
                height="1.5em"
                fill="currentColor"
                aria-hidden="true"
              >
                <path d="M988 548c-19.9 0-36-16.1-36-36 0-59.4-11.6-117-34.6-171.3a440.45 440.45 0 0 0-94.3-139.9 437.71 437.71 0 0 0-139.9-94.3C629 83.6 571.4 72 512 72c-19.9 0-36-16.1-36-36s16.1-36 36-36c69.1 0 136.2 13.5 199.3 40.3C772.3 66 827 103 874 150c47 47 83.9 101.8 109.7 162.7 26.7 63.1 40.2 130.2 40.2 199.3.1 19.9-16 36-35.9 36z" />
              </svg>
            </i>
            <span />
          </p>
        }
      </section>
      : ''
    )
  }
})

export default cropperLoading
```

---

## 小結

| 概念 | 說明 |
|------|------|
| `.tsx` 元件 | 邏輯和結構寫在一起，適合簡單元件 |
| `defineComponent` | `.tsx` 裡定義 Vue 元件的方式 |
| `render()` | 回傳 JSX，等同 `<template>` |
| `$slots.default()` | 在 JSX 裡渲染插槽內容 |
| `fill="currentColor"` | SVG 繼承父元素 CSS color |

下一篇：[12 — lib/style/index.scss（樣式設計）](./12-styles.md)
