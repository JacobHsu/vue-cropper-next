# 10 — lib/filter/index.ts（Canvas 濾鏡原理）

## Canvas 的像素操作原理

在瀏覽器裡，Canvas 可以讓你讀取和修改圖片的**每一個像素**。

每個像素由 4 個數值組成（RGBA）：

| 通道 | 意義 | 範圍 |
|------|------|------|
| R（Red）| 紅色強度 | 0（無）~ 255（最強）|
| G（Green）| 綠色強度 | 0 ~ 255 |
| B（Blue）| 藍色強度 | 0 ~ 255 |
| A（Alpha）| 透明度 | 0（全透明）~ 255（不透明）|

一張 100x100 的圖片，共有 10000 個像素，也就是 40000 個數值（4 × 10000）。

`ctx.getImageData()` 可以取得這些數值，`ctx.putImageData()` 可以把修改後的數值寫回。

---

## 建立 lib/filter/index.ts

### grayscale：灰階濾鏡

```ts
const grayscale = (canvas: HTMLCanvasElement): HTMLCanvasElement => {
  let gray: number = 0
  const ctx = canvas.getContext('2d') as CanvasRenderingContext2D
  let imgData: any = []
  
  try {
    imgData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  } catch (e) {
    console.error(e)
    return canvas
  }
  
  for (let i = 0; i < canvas.width * canvas.height * 4; i += 4) {
    const r: number = imgData.data[i]
    const g: number = imgData.data[i + 1]
    const b: number = imgData.data[i + 2]
    // 人眼感知亮度的加權公式
    gray = ~~(0.2989 * r + 0.587 * g + 0.114 * b)
    imgData.data[i] = gray
    imgData.data[i + 1] = gray
    imgData.data[i + 2] = gray
    // imgData.data[i + 3] = alpha，不修改透明度
  }
  
  ctx.putImageData(imgData, 0, 0)
  return canvas
}
```

**核心邏輯**：`0.2989 * r + 0.587 * g + 0.114 * b`

這不是簡單的平均（`(r + g + b) / 3`），而是根據**人眼對各顏色的敏感度**加權：
- 人眼對**綠色**最敏感（0.587）
- 對**紅色**次之（0.2989）
- 對**藍色**最不敏感（0.114）

所以綠色通道的權重最高，讓灰階結果更符合人眼感知的亮度。

**`~~` 運算子**：雙波浪號，等同於 `Math.floor()`，把浮點數截成整數（像素值必須是整數）。

**逐步解析 `for` 迴圈**：

```
i=0:  像素[0] 的 R、G、B、A → 索引 0, 1, 2, 3
i=4:  像素[1] 的 R、G、B、A → 索引 4, 5, 6, 7
i=8:  像素[2] 的 R、G、B、A → 索引 8, 9, 10, 11
...
```

每次 `i += 4`，跳過一整個像素的 4 個通道。

---

### blackAndWhite：黑白濾鏡（高對比灰階）

```ts
const blackAndWhite = (canvas: HTMLCanvasElement): HTMLCanvasElement => {
  const ctx = canvas.getContext('2d') as CanvasRenderingContext2D
  let imgData: any = []
  
  try {
    imgData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  } catch (e) {
    console.error(e)
    return canvas
  }
  
  let gray: number = 0
  let i = 0
  while (i < canvas.width * canvas.height * 4) {
    const r = imgData.data[i]
    const g = imgData.data[i + 1]
    const b = imgData.data[i + 2]
    // 若亮度 > 128：全白（255），否則全黑（0）
    gray = ~~(0.2989 * r + 0.587 * g + 0.114 * b) > 128 ? 255 : 0
    imgData.data[i] = gray
    imgData.data[i + 1] = gray
    imgData.data[i + 2] = gray
    i += 4
  }
  
  ctx.putImageData(imgData, 0, 0)
  return canvas
}
```

和灰階的差別：灰階保留所有亮度層次（0~255），黑白只有兩個值（0 或 255）。

閾值 128 是一個經驗值（灰色的中間點）。

---

### oldPhoto：老照片濾鏡（棕褐色調）

```ts
const oldPhoto = (canvas: HTMLCanvasElement): HTMLCanvasElement => {
  const ctx = canvas.getContext('2d') as CanvasRenderingContext2D
  let imgData: any = []
  
  try {
    imgData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  } catch (e) {
    console.error(e)
    return canvas
  }
  
  let i = 0
  while (i < canvas.width * canvas.height * 4) {
    const r = imgData.data[i]
    const g = imgData.data[i + 1]
    const b = imgData.data[i + 2]
    imgData.data[i]     = 0.393 * r + 0.769 * g + 0.189 * b  // 新 R
    imgData.data[i + 1] = 0.349 * r + 0.686 * g + 0.168 * b  // 新 G
    imgData.data[i + 2] = 0.272 * r + 0.543 * g + 0.131 * b  // 新 B
    i += 4
  }
  
  ctx.putImageData(imgData, 0, 0)
  return canvas
}
```

老照片效果的原理：把 RGB 重新混合，讓紅色通道偏高（偏暖），藍色通道偏低（減少冷色），產生棕褐色的「復古感」。

這組係數（0.393、0.769 等）是業界廣泛使用的棕褐色（Sepia）矩陣，來自 W3C 的濾鏡規範。

---

## 濾鏡的函數簽名設計

注意三個濾鏡函數都有相同的簽名：

```ts
(canvas: HTMLCanvasElement) => HTMLCanvasElement
```

這個設計很重要。在 `vue-cropper.vue` 的 `filter` prop 型別是：

```ts
filter?: ((canvas: HTMLCanvasElement) => HTMLCanvasElement) | null
```

使用者可以傳入**任意**符合這個簽名的函數，包括自訂濾鏡：

```html
<VueCropper :filter="grayscale" />
<VueCropper :filter="myCustomFilter" />
```

這是一個**開放式介面**設計：函式庫提供幾個內建濾鏡，但使用者可以寫任意複雜的濾鏡，只要接受一個 canvas、回傳一個 canvas 即可。

---

## 導出

```ts
export { grayscale, blackAndWhite, oldPhoto }
```

使用者可以這樣使用：

```ts
import { grayscale, blackAndWhite, oldPhoto } from 'cropper-next-vue'

// 或自訂濾鏡
const myFilter = (canvas) => {
  const ctx = canvas.getContext('2d')
  // 你的像素操作...
  return canvas
}
```

---

## 完整的 lib/filter/index.ts

```ts
// 灰階濾鏡
const grayscale = (canvas: HTMLCanvasElement): HTMLCanvasElement => {
  let gray: number = 0
  const ctx: CanvasRenderingContext2D = canvas.getContext('2d') as CanvasRenderingContext2D
  let imgData: any = []
  try {
    imgData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  } catch (e) {
    console.error(e)
    return canvas
  }
  for (let i = 0; i < canvas.width * canvas.height * 4; i += 4) {
    const r: number = imgData.data[i]
    const g: number = imgData.data[i + 1]
    const b: number = imgData.data[i + 2]
    gray = ~~(0.2989 * r + 0.587 * g + 0.114 * b)
    imgData.data[i] = gray
    imgData.data[i + 1] = gray
    imgData.data[i + 2] = gray
  }
  ctx.putImageData(imgData, 0, 0)
  return canvas
}

// 黑白
const blackAndWhite = (canvas: HTMLCanvasElement): HTMLCanvasElement => {
  const ctx: CanvasRenderingContext2D = canvas.getContext('2d') as CanvasRenderingContext2D
  let imgData: any = []
  try {
    imgData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  } catch (e) {
    console.error(e)
    return canvas
  }
  let gray: number = 0
  let i = 0
  while (i < canvas.width * canvas.height * 4) {
    const r: number = imgData.data[i]
    const g: number = imgData.data[i + 1]
    const b: number = imgData.data[i + 2]
    gray = ~~(0.2989 * r + 0.587 * g + 0.114 * b) > 128 ? 255 : 0
    imgData.data[i] = gray
    imgData.data[i + 1] = gray
    imgData.data[i + 2] = gray
    i += 4
  }
  ctx.putImageData(imgData, 0, 0)
  return canvas
}

// 老照片算法
const oldPhoto = (canvas: HTMLCanvasElement): HTMLCanvasElement => {
  const ctx: CanvasRenderingContext2D = canvas.getContext('2d') as CanvasRenderingContext2D
  let imgData: any = []
  try {
    imgData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  } catch (e) {
    console.error(e)
    return canvas
  }
  let i = 0
  while (i < canvas.width * canvas.height * 4) {
    const r: number = imgData.data[i]
    const g: number = imgData.data[i + 1]
    const b: number = imgData.data[i + 2]
    imgData.data[i] = 0.393 * r + 0.769 * g + 0.189 * b
    imgData.data[i + 1] = 0.349 * r + 0.686 * g + 0.168 * b
    imgData.data[i + 2] = 0.272 * r + 0.543 * g + 0.131 * b
    i += 4
  }
  ctx.putImageData(imgData, 0, 0)
  return canvas
}

export { grayscale, blackAndWhite, oldPhoto }
```

---

## 小結

這三個濾鏡展示了 Canvas 像素操作的基本模式：

1. `getImageData()` → 取得像素陣列
2. 遍歷每個像素（`i += 4`）→ 讀取 R、G、B
3. 計算新的 R、G、B → 寫回
4. `putImageData()` → 更新畫面

掌握這個模式後，你可以實現任意複雜的影像處理效果：模糊、銳化、色彩校正、邊緣偵測……這是整個圖像處理領域的基礎。

下一篇：[11 — lib/loading.tsx（JSX 元件寫法）](./11-loading.md)
