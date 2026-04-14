# 17 — lib/vue-cropper.vue Part 5：Template 結構與生命週期

## Template 全覽

```html
<template>
  <section class="vue-cropper" ...>

    <!-- 圖片層（有圖片時才渲染）-->
    <section v-if="imgs" class="cropper-box cropper-fade-in">
      <!-- 圖片 canvas -->
      <section class="cropper-box-canvas" :style="LayoutContainer.imgExhibitionStyle">
        <img :src="imgs" alt="cropper-next-vue" />
      </section>
      <!-- 拖曳感應層 -->
      <section :class="computedClassDrag()" ref="cropperImg" />
    </section>

    <!-- 裁切框（條件顯示）-->
    <section v-if="cropping && imgs && shouldShowCropBox"
      class="cropper-crop-box cropper-fade-in"
      :style="getCropBoxStyle()">
      <!-- 裁切框的視覺框線 -->
      <span class="cropper-view-box" :style="{outlineColor: cropColor}">
        <img v-if="img" :src="imgs" :style="getCropImgStyle()" alt="cropper-img" />
      </span>
      <!-- 裁切框的事件感應層 -->
      <span class="cropper-face cropper-move" ref="cropperBox" />
    </section>

    <!-- 拖曳提示 -->
    <section v-if="isDrag" class="drag">
      <slot name="drag">
        <p>拖曳圖片到此</p>
      </slot>
    </section>

    <!-- Loading -->
    <cropperLoading :is-visible="imgLoading">
      <template v-if="slots.loading" #default>
        <slot name="loading"></slot>
      </template>
    </cropperLoading>

  </section>
</template>
```

---

## 逐段解析

### 外層容器

```html
<section
  class="vue-cropper"
  :style="wrapperStyle"          ← 動態尺寸和樣式
  ref="cropperRef"               ← 取得 DOM 元素（讀取實際尺寸用）
  :onMouseover="mouseInCropper"  ← 滑鼠進入：開始監聽滾輪
  :onMouseout="mouseOutCropper"  ← 滑鼠離開：停止監聽滾輪
>
```

**為什麼用 `:onMouseover` 而不是 `@mouseover`？**

在 Vue 3 裡，`:onMouseover` 和 `@mouseover` 在大多數情況下是等效的。用 `:onMouseover` 是一種更靠近 DOM 屬性的寫法，在某些邊界情況下更可預期。

---

### 圖片層：.cropper-box

```html
<section v-if="imgs" class="cropper-box cropper-fade-in">
```

**`v-if="imgs"`**：只有在圖片 URL（Blob URL）存在時才渲染。

圖片載入中時 `imgs.value = ''`，圖片層不存在，顯示 loading。

**`cropper-fade-in`**：CSS 淡入動畫，讓圖片出現時有平滑效果。

---

### 圖片本體

```html
<section
  class="cropper-box-canvas"
  :style="LayoutContainer.imgExhibitionStyle"
>
  <img :src="imgs" alt="cropper-next-vue" />
</section>
```

**為什麼不直接對 `<img>` 套用 transform，而是套在外層 `<section>` 上？**

因為 `<img>` 的 `transform` 在某些瀏覽器版本會有渲染問題（子元素繼承 transform）。

把 transform 套在容器上，`<img>` 只負責顯示圖片，不參與定位計算，更穩定。

---

### 拖曳感應層

```html
<section :class="computedClassDrag()" ref="cropperImg" />
```

```ts
const computedClassDrag = (): string => {
  const className = ['cropper-drag-box']
  if (cropping.value) {
    className.push('cropper-modal')  // 加上半透明遮罩
  }
  return className.join(' ')
}
```

這個元素是**完全透明**的，它的作用是捕捉拖曳事件（`mousedown`/`touchstart`）。

**為什麼要用獨立的透明層，而不是直接在圖片上捕捉事件？**

圖片元素有自己的拖曳預設行為（瀏覽器會試圖「拖走」圖片），用透明層覆蓋可以避免這個問題，也讓事件監聽更乾淨。

---

### 裁切框層

```html
<!-- lib/vue-cropper.vue -->
<section
  v-if="cropping && imgs && shouldShowCropBox"
  class="cropper-crop-box cropper-fade-in"
  :style="getCropBoxStyle()"
>
```

三個條件都要滿足才顯示：
1. `cropping`：沒有呼叫 `clearCrop()` 清除裁切框
2. `imgs`：圖片已載入
3. `shouldShowCropBox`：裁切框比容器小（如果裁切框 = 容器大小，就沒有意義顯示）

**`:style="getCropBoxStyle()"`**

注意這裡直接呼叫函數，而不是用 `computed`。

原因：`getCropBoxStyle` 有**副作用**——它除了計算樣式，還會把結果寫入 `LayoutContainer.cropExhibitionStyle.div`（供外部使用）。

因為這個副作用，不能用 `computed`（computed 應該是純函數）。

Vue 在每次重新渲染時都會重新呼叫 template 裡的函數，所以樣式始終是最新的。

---

### 裁切框內部圖片

```html
<span class="cropper-view-box" :style="{outlineColor: cropColor}">
  <img v-if="img" :src="imgs" :style="getCropImgStyle()" alt="cropper-img" />
</span>
```

**`outlineColor: cropColor`**：裁切框的邊框顏色可以由 `cropColor` prop 控制。

**`v-if="img"` vs 外層的 `v-if="imgs"`**：

- `imgs`：Blob URL（內部使用）
- `img`：使用者傳入的 prop URL

當 `img` prop 為空字串時，不渲染這張圖片。這是一個邊緣情況的保護。

---

### 拖曳上傳提示

```html
<section v-if="isDrag" class="drag">
  <slot name="drag">
    <p>拖曳圖片到此</p>
  </slot>
</section>
```

**slot 的使用**：

提供預設內容（`<p>拖曳圖片到此</p>`），但允許使用者自訂：

```html
<VueCropper>
  <template #drag>
    <div class="my-drag-hint">
      <img src="upload-icon.svg" />
      <p>放開即可上傳</p>
    </div>
  </template>
</VueCropper>
```

---

### Loading 元件

```html
<cropperLoading :is-visible="imgLoading">
  <template v-if="slots.loading" #default>
    <slot name="loading"></slot>
  </template>
</cropperLoading>
```

```ts
const slots = useSlots()
```

**`useSlots()`**：取得父元件傳入的插槽物件。

這段邏輯：
- 如果父元件提供了 `#loading` 插槽 → 使用自訂 loading
- 否則 → 使用 `cropperLoading` 的預設 SVG 旋轉動畫

**傳遞插槽的技巧**：

父元件傳給 `<VueCropper #loading>` 的 slot，在 `vue-cropper.vue` 內被接收，再轉手傳給 `<cropperLoading #default>`。這就是「插槽透傳」（slot forwarding）。

---

## 生命週期

### onMounted：初始化

```ts
onMounted(() => {
  if (props.img) {
    checkedImg(props.img)  // 如果有初始圖片，立刻載入
  } else {
    imgs.value = ''
  }

  // 綁定拖曳上傳事件
  const dom = cropperRef.value
  dom.addEventListener('dragover', dragover, false)
  dom.addEventListener('dragend', dragend, false)
  dom.addEventListener('drop', drop, false)
})
```

`onMounted` 是在 DOM 插入後執行的，所以這裡可以安全地：
- 讀取 `cropperRef.value`（DOM 元素已存在）
- 呼叫 `checkedImg`（需要讀取容器尺寸）

---

### onUnmounted：清理

```ts
onUnmounted(() => {
  // 取消所有進行中的 requestAnimationFrame
  if (realTimeFrame) {
    cancelAnimationFrame(realTimeFrame)
    realTimeFrame = 0
  }

  // 移除拖曳上傳事件
  cropperRef.value?.removeEventListener('drop', drop, false)
  cropperRef.value?.removeEventListener('dragover', dragover, false)
  cropperRef.value?.removeEventListener('dragend', dragend, false)

  // 移除拖曳事件
  unbindMoveImg()
  unbindMoveCrop()
})
```

**為什麼清理這麼重要？**

Vue 元件銷毀時，Vue 會自動移除 `@click` 等綁定在 template 元素上的事件。

但是：
- `window.addEventListener` 綁在全域 window 上，不在 Vue 的管轄範圍，**不會自動移除**
- `requestAnimationFrame` 的回調，也**不會自動取消**

如果不手動清理，元件銷毀後，這些事件監聽和動畫回調仍然活著，繼續消耗記憶體，並且在回調執行時可能試圖更新已銷毀的元件，造成錯誤。

**`?.` 可選鏈（Optional Chaining）**：

`cropperRef.value?.removeEventListener(...)` = 如果 `cropperRef.value` 是 null/undefined，直接跳過，不拋出錯誤。

---

## vue-cropper.vue 完整架構回顧

```
vue-cropper.vue
│
├── Script Setup
│   ├── Imports（Vue API + 所有模組）
│   ├── Props 定義（defineProps + withDefaults）
│   ├── DOM refs（cropperRef, cropperImg, cropperBox）
│   ├── Events（defineEmits）
│   ├── 核心狀態（LayoutContainer, imgs, canvas...）
│   ├── Computed（wrapperStyle, effectiveCropLayoutStyle...）
│   ├── Watch（img, imgs, filter, mode, wrapperStyle...）
│   ├── 圖片載入函數（checkedImg, renderFilter, createImg...）
│   ├── 互動函數（moveImg, setScale, reboundImg, setRotate...）
│   ├── 裁切框函數（renderCrop, moveCrop, getCropBoxStyle...）
│   ├── 導出函數（getCropData, getCropBlob）
│   ├── 生命週期（onMounted, onUnmounted）
│   └── defineExpose（開放公開 API）
│
└── Template
    ├── .vue-cropper（外層容器）
    ├── .cropper-box（圖片層）
    │   ├── .cropper-box-canvas（圖片）
    │   └── .cropper-drag-box（拖曳感應層）
    ├── .cropper-crop-box（裁切框層）
    │   ├── .cropper-view-box（裁切框視覺框線）
    │   │   └── img（裁切框內圖片）
    │   └── .cropper-face（裁切框事件層）
    ├── .drag（拖曳上傳提示）
    └── cropperLoading（Loading 元件）
```

---

## 小結

Template 的設計核心是「多層疊加」：所有子層都是 `position: absolute` 並填滿容器，按需顯示或隱藏。

生命週期的設計核心是「對稱清理」：`onMounted` 綁定的每一個事件，都必須在 `onUnmounted` 解綁。

到這裡，`vue-cropper.vue` 的五個部分全部說明完畢。

下一篇：[18 — lib/index.ts + lib/typings/index.d.ts（入口與型別聲明）](./18-entry-types.md)
