# 13 — lib/vue-cropper.vue Part 1：Props 定義與狀態結構

## 這篇的範圍

`vue-cropper.vue` 是整個函式庫的核心，把前面十幾個模組整合在一起。

由於程式碼較長（1066 行），我們分五篇說明：

| 篇章 | 內容 |
|------|------|
| **Part 1（本篇）** | imports、Props 定義、狀態資料結構 |
| Part 2 | 圖片加載流程 |
| Part 3 | 拖曳、縮放、回彈邏輯 |
| Part 4 | 裁切框操作與導出 |
| Part 5 | Template 結構與生命週期 |

---

## imports

```ts
import { computed, nextTick, onMounted, onUnmounted, reactive, ref, toRefs, watch, useSlots } from 'vue'
```

從 Vue 引用所有用到的 Composition API 函數。（這些都在 [02 — Vue 3 快速入門](./02-vue3-primer.md) 說明過了。）

```ts
import type {
  InterfaceAxis,
  InterfaceImgLoad,
  InterfaceLayout,
  InterfaceLayoutInput,
  InterfaceLength,
  InterfaceLayoutStyle,
  InterfaceMessageEvent,
  InterfaceModeHandle,
  InterfaceRealTimePreview,
  InterfaceTransformStyle,
} from './interface'
```

引用型別定義。`import type` 只引用型別，不影響執行期。

```ts
import {
  loadImg, getExif, resetImg, createImgStyle, translateStyle,
  loadFile, getCropImgData, detectionBoundary, setAnimation, checkOrientationImage,
} from './common'
import { supportWheel, changeImgSize, isIE, changeImgSizeByTouch } from './changeImgSize'
import { BOUNDARY_DURATION, RESISTANCE } from './config'
import TouchEvent from './touch'
import cropperLoading from './loading'
import './style/index.scss'  // 直接引入 CSS，Vite 會處理
```

---

## Props 定義

### 宣告 interface

```ts
interface InterfaceVueCropperProps {
  img?: string              // 圖片 URL（可選，動態也可以）
  wrapper?: InterfaceLayout // 外層容器的寬高和樣式
  cropLayout?: InterfaceLayoutInput // 裁切框的寬高
  color?: string            // 主題色（裁切框顏色等）
  filter?: ((canvas: HTMLCanvasElement) => HTMLCanvasElement) | null // 濾鏡函數
  outputType?: string       // 輸出格式（'png'、'jpeg'、'webp'）
  outputSize?: number       // 輸出品質（0~1）
  full?: boolean            // 高清導出（使用 devicePixelRatio）
  original?: boolean        // 使用圖片原始像素尺寸輸出
  maxSideLength?: number    // 輸出圖片最長邊限制
  mode?: keyof InterfaceModeHandle // 圖片佈局模式
  cropColor?: string        // 裁切框邊框顏色
  defaultRotate?: number    // 初始旋轉角度
  centerBox?: boolean       // 圖片是否限制在裁切框內（不能拖出去）
  centerWrapper?: boolean   // 圖片是否限制在容器內
  centerBoxDelay?: number   // 裁切框邊界回彈時間（ms）
  centerWrapperDelay?: number // 容器邊界回彈時間（ms）
}
```

> **TypeScript 穿插說明 — 聯合型別**  
> `((canvas: HTMLCanvasElement) => HTMLCanvasElement) | null`  
> 這個型別表示：這個值可以是一個函數，或者是 `null`（不套用濾鏡）。  
> `|` 是「或」，`null` 表示明確傳入「沒有值」。  
> 為什麼要明確寫 `| null` 而不是用 `?`（可選）？  
> 因為 `?` 表示「可以不傳」，但傳入 `null` 表示「明確告訴我不要用濾鏡」，語意不同。

### 設定預設值

```ts
const props = withDefaults(defineProps<InterfaceVueCropperProps>(), {
  img: '',
  wrapper: () => ({ width: 300, height: 300 }),
  cropLayout: () => ({ width: 200, height: 200 }),
  color: '#fff',
  filter: null,
  outputType: 'png',
  outputSize: 1,
  full: true,
  original: false,
  maxSideLength: 3000,
  mode: 'cover',
  cropColor: '#fff',
  defaultRotate: 0,
  centerBox: false,
  centerWrapper: false,
  centerBoxDelay: BOUNDARY_DURATION,
  centerWrapperDelay: BOUNDARY_DURATION,
})
```

**為什麼 `wrapper` 和 `cropLayout` 的預設值用函數 `() => ({...})`？**

在 Vue 3 中，如果 Props 的預設值是物件或陣列，必須用**工廠函數**（回傳物件的函數），而不是直接寫物件。

原因：如果直接寫 `wrapper: { width: 300, height: 300 }`，所有元件實例會**共用同一個物件**，修改一個實例的 wrapper 會影響其他實例。工廠函數確保每個實例都有自己的物件。

---

## 狀態資料結構

### DOM 元素引用

```ts
const cropperRef = ref()   // 外層容器（.vue-cropper）
const cropperImg = ref()   // 拖曳感應層（.cropper-drag-box）
const cropperBox = ref()   // 裁切框的可點擊層（.cropper-face）
```

這三個 `ref` 在 template 裡用 `ref="cropperRef"` 等語法綁定到對應的 DOM 元素。

---

### 事件相關

```ts
const emit = defineEmits<{
  (e: 'img-load', obj: InterfaceImgLoad): void
  (e: 'img-upload', url: string): void
  (e: 'real-time', payload: InterfaceRealTimePreview): void
  (e: 'realTime', payload: InterfaceRealTimePreview): void
}>()
```

四個事件：

| 事件 | 觸發時機 | 父元件的處理 |
|------|---------|------------|
| `img-load` | 圖片載入成功/失敗 | 顯示錯誤訊息 |
| `img-upload` | 拖曳檔案上傳 | 可以做額外驗證 |
| `real-time` | 任何操作（拖曳/縮放/旋轉）| 即時更新預覽圖 |
| `realTime` | 同上（駝峰式命名，向下相容）| 同上 |

---

### 核心佈局狀態：LayoutContainer

```ts
const LayoutContainer = reactive({
  // 圖片真實寬高（載入後獲取）
  imgLayout: {
    width: 0,
    height: 0,
  },
  // 外層容器寬高（從 DOM 讀取）
  wrapLayout: {
    width: 0,
    height: 0,
  },
  // 圖片的座標 + 縮放 + 旋轉（全部在這裡）
  imgAxis: {
    x: 0,
    y: 0,
    scale: 0,
    rotate: 0,
  },
  // 圖片的 CSS 樣式（由 translateStyle 計算得出）
  imgExhibitionStyle: {
    width: '',
    height: '',
    transform: '',
  },
  // 裁切框左上角的座標
  cropAxis: {
    x: 0,
    y: 0,
  },
  // 裁切框的 CSS 樣式（外層 div 和內部 img 各一份）
  cropExhibitionStyle: {
    div: {},
    img: {},
  }
})
```

**設計重點**：所有「佈局相關的狀態」集中在一個 `reactive` 物件裡，而不是分散成 12 個 `ref`。

好處：
- 一眼看出「佈局相關的東西全在 LayoutContainer」
- 傳遞給子函數時更方便（`{ ...LayoutContainer.imgAxis }` 而不是傳 4 個參數）
- 未來擴充（加新的佈局相關狀態）只需在這個物件裡加，不用到處宣告

---

### 其他狀態

```ts
const imgLoading = ref(false)   // 是否顯示 loading
const imgs = ref('')            // 圖片的當前 Blob URL
let canvas: HTMLCanvasElement | null = null  // 已校正方向的 canvas

const isDrag = ref(false)       // 是否正在拖曳（用來顯示拖曳提示）
const cropping = ref(true)      // 是否顯示裁切框

let cropImg: TouchEvent | null = null   // 圖片的拖曳事件實例
let cropBox: TouchEvent | null = null   // 裁切框的拖曳事件實例

const setWaitFunc = ref<ReturnType<typeof window.setTimeout> | null>(null)
const isImgTouchScale = ref(false)  // 是否正在雙指縮放（縮放時禁止裁切框移動）
```

> **TypeScript 穿插說明 — `ReturnType<typeof ...>`**  
> `ReturnType<typeof window.setTimeout>` 取得 `setTimeout` 函數的**回傳值型別**。  
> `setTimeout` 回傳的是一個計時器 ID（在 Node.js 是 `NodeJS.Timeout`，在瀏覽器是 `number`）。  
> 用 `ReturnType` 而不是直接寫 `number`，是因為這樣在 Node.js 和瀏覽器環境都能正確推斷型別。

**為什麼 `canvas` 用 `let` 而不是 `ref`？**

`canvas` 不需要是響應式的。它是一個內部的工作物件，不需要在模板裡渲染，也不需要 Vue 追蹤它的變化。用普通的 `let` 宣告，避免不必要的響應式開銷。

**為什麼 `cropImg` 和 `cropBox` 用 `let` 而不是 `ref`？**

同理，`TouchEvent` 的實例只在 `bindMoveImg`/`bindMoveCrop` 時建立，在 `unbind` 時設為 `null`。這是純粹的物件管理，不需要響應式。

---

### toRefs：解構 Props

```ts
const {
  img, filter, mode, defaultRotate, outputType, outputSize,
  full, original, maxSideLength, centerBox, cropLayout,
  centerWrapper, centerBoxDelay, centerWrapperDelay,
} = toRefs(props)
```

`toRefs(props)` 把整個 props 物件轉成各個 `ref`，讓每個 prop 可以在 `watch` 裡單獨監聽：

```ts
watch(img, (val) => {
  // img 是 Ref<string>，可以被 watch
})
```

如果不用 `toRefs`，`props.img` 只是一個普通值，Vue 不會追蹤它的變化（因為它不是在 `watch` 的依賴收集範圍內）。

---

### computed：尺寸計算

```ts
// 容器樣式（數字轉字串 + 單位）
const wrapperStyle = computed(() => ({
  ...props.wrapper,
  width: normalizeLengthStyle(props.wrapper.width),
  height: normalizeLengthStyle(props.wrapper.height),
}))

// 裁切框尺寸（字串轉數字）
const cropLayoutStyle = computed(() => ({
  width: parseLength(props.cropLayout.width),
  height: parseLength(props.cropLayout.height),
}))

// 有效裁切框尺寸（不能超過容器）
const effectiveCropLayoutStyle = computed(() => {
  const wrapWidth = LayoutContainer.wrapLayout.width
  const wrapHeight = LayoutContainer.wrapLayout.height
  const cropWidth = cropLayoutStyle.value.width
  const cropHeight = cropLayoutStyle.value.height
  return {
    width: wrapWidth > 0 ? Math.min(cropWidth, wrapWidth) : cropWidth,
    height: wrapHeight > 0 ? Math.min(cropHeight, wrapHeight) : cropHeight,
  }
})

// 是否應該顯示裁切框（裁切框比容器小才顯示）
const shouldShowCropBox = computed(() => {
  const wrapWidth = LayoutContainer.wrapLayout.width
  const wrapHeight = LayoutContainer.wrapLayout.height
  if (wrapWidth <= 0 || wrapHeight <= 0) return true
  return (
    effectiveCropLayoutStyle.value.width < wrapWidth ||
    effectiveCropLayoutStyle.value.height < wrapHeight
  )
})
```

**`normalizeLengthStyle`**：把 `300`（數字）轉成 `'300px'`（字串），把 `'50%'`（字串）直接回傳。

**`parseLength`**：把 `'300px'` 轉成 `300`（數字），把 `300` 直接回傳。

---

## 小結

這一篇建立了主元件的「骨架」：

1. **Props**：使用者可以傳入的所有設定項
2. **LayoutContainer**：所有佈局狀態集中管理
3. **computed**：從 Props/狀態衍生的計算值

接下來的四篇會逐步填入「肉」（函數邏輯）。

下一篇：[14 — lib/vue-cropper.vue Part 2：圖片加載流程](./14-main-comp-2-img-load.md)
