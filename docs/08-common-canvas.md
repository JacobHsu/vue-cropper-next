# 08 — lib/common.ts 後段（Canvas 操作、邊界計算、動畫）

## 這篇涵蓋的內容

`common.ts` 後段包含三組功能：

1. **Canvas 操作**：把圖片畫進 canvas，再輸出裁切結果
2. **邊界計算**：判斷圖片是否超出允許範圍，計算要回彈到哪裡
3. **緩動動畫**：讓回彈有平滑的動畫效果

---

## 一、Canvas 操作

### loadFile：從本機檔案載入圖片

```ts
export const loadFile = async (file: File): Promise<any> => {
  if (!file) {
    return ''
  }
  if (!/\.(gif|jpg|jpeg|png|bmp|GIF|JPG|PNG|WEBP|webp)$/.test(file.name)) {
    alert('图片类型必须是.gif,jpeg,jpg,png,bmp,webp中的一种')
    return ''
  }
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = (event: Event) => {
      let data: string = ''
      const targetTwo = event.target as FileReader
      if (typeof targetTwo.result === 'object' && targetTwo.result) {
        data = window.URL.createObjectURL(new Blob([targetTwo.result]))
      } else {
        data = targetTwo.result as string
      }
      resolve(data)
    }
    reader.onerror = reject
    reader.readAsArrayBuffer(file)  // 讀成 ArrayBuffer
  })
}
```

**整個流程**：使用者拖曳圖片進裁剪元件 → 觸發 `drop` 事件 → 呼叫 `loadFile(file)` → 讀成 ArrayBuffer → 建立 Blob URL → 回傳 URL 字串

**為什麼讀成 ArrayBuffer 再建立 Blob URL，而不直接用 `readAsDataURL`（Base64）？**

- Blob URL 是指向記憶體的指標（`blob:http://...`），非常短
- Base64 字串是把整個圖片編碼成文字，一張 1MB 的圖片變成約 1.37MB 的字串
- 大圖片用 Base64 會讓記憶體和 DOM 都很沉重

> **TypeScript 穿插說明 — `as` 型別斷言**  
> `event.target as FileReader` 告訴 TypeScript「我知道這個 `event.target` 是 `FileReader`，相信我」。  
> `as` 不會改變執行期的行為，只影響 TypeScript 的型別推斷。  
> 在確定型別的情況下使用，不確定時應該用 `instanceof` 判斷。

---

### getImgCanvas：把圖片畫進 Canvas（支援旋轉）

```ts
export const getImgCanvas = (
  img: HTMLImageElement,
  imgLayout: InterfaceLayoutStyle,
  rotate: number = 0,
  scale: number = 1,
): HTMLCanvasElement => {
  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d') as CanvasRenderingContext2D
  
  let { width, height } = imgLayout
  let dx = 0
  let dy = 0
  let max = 0

  width = width * scale
  height = height * scale
  
  canvas.width = width
  canvas.height = height
```

這個函數建立一個新的 canvas，並把圖片按指定尺寸畫進去。

**旋轉時的特殊處理**：

```ts
  if (rotate) {
    // 旋轉後的圖片外接圓直徑 = 對角線長度
    max = Math.ceil(Math.sqrt(width * width + height * height))
    canvas.width = max
    canvas.height = max
    ctx.translate(max / 2, max / 2)   // 把原點移到中心
    ctx.rotate((rotate * Math.PI) / 180)
    dx = -max / 2 + (max - width) / 2
    dy = -max / 2 + (max - height) / 2
  }

  ctx.drawImage(img, dx, dy, width, height)
  ctx.restore()
  return canvas
}
```

**為什麼旋轉後 canvas 要放大成對角線大小？**

```
  旋轉前（300x200）：
  ┌─────────────┐
  │             │
  └─────────────┘
  
  旋轉 45°：
      ╱─────╲
     ╱       ╲
    ╱         ╲
    ╲         ╱
     ╲       ╱
      ╲─────╱
  
  外接圓直徑 = √(300² + 200²) ≈ 361
```

旋轉後圖片的角落會超出原來的範圍，如果 canvas 不夠大，角落會被裁掉。使用對角線長度作為 canvas 的邊長，確保旋轉後整張圖片都在 canvas 裡。

之後的裁切計算會減去這個多出來的空白空間。

---

### getCropImgData：生成最終裁切結果

這是整個函式庫最複雜的函數，負責生成使用者最終要的裁切圖片。

#### 參數說明

```ts
interface CropImageOptions {
  type?: 'base64' | 'blob'    // 輸出格式
  outputType: string           // 圖片格式（png/jpeg/webp）
  outputSize?: number          // 品質（0~1）
  full?: boolean               // 是否高清（使用 devicePixelRatio）
  original?: boolean           // 是否使用原始像素尺寸（忽略縮放）
  maxSideLength?: number       // 最長邊不超過多少像素
  url: string                  // 圖片的 Blob URL
  imgAxis: InterfaceImgAxis    // 圖片的座標和縮放/旋轉狀態
  imgLayout: InterfaceLayoutStyle  // 圖片原始尺寸
  cropLayout: InterfaceLayoutStyle // 裁切框尺寸
  cropAxis: InterfaceAxis      // 裁切框位置
  cropping: boolean            // 是否有啟用裁切框
}
```

#### 座標系換算

這是整個函數最難理解的部分。

裁剪元件的「螢幕座標」和「圖片原始座標」是不同的，因為圖片可能被縮放了（scale ≠ 1）。

```
螢幕座標（使用者看到的）  ──  ÷ scale  ──→  圖片原始座標
```

`original` 模式（輸出原始尺寸）：
- `coordinateScale = 1 / axisScale`（螢幕座標 → 原始座標）
- `canvasScale = 1`（canvas 大小 = 原始圖片大小）

一般模式（高清輸出）：
- `coordinateScale = 1`（螢幕座標直接用）
- `canvasScale = max(1, scale)`（canvas 放大，提高解析度）

```ts
const axisScale = imgAxis.scale > 0 ? imgAxis.scale : 1
const coordinateScale = original ? 1 / axisScale : 1
const canvasScale = original ? 1 : Math.max(1, axisScale)
const drawScale = original ? 1 : axisScale / canvasScale
```

#### 高清導出（devicePixelRatio）

```ts
const pixelRatio = full ? Math.max(1, window.devicePixelRatio || 1) : 1
```

Retina 螢幕（iPhone、MacBook）的 `devicePixelRatio` 是 2 或 3。

這表示 1 個 CSS 像素 = 4 個（2×2）或 9 個（3×3）實體像素。

如果不處理：導出的圖片在 Retina 螢幕上會看起來模糊（因為 CSS 100x100 的裁切框對應到 200x200 的實體像素，但你輸出的只有 100x100）。

**加上 devicePixelRatio 之後**：導出 200x200 的圖片，在 Retina 螢幕上清晰顯示。

#### maxSideLength 限制

```ts
const maxEdge = Math.max(width * renderRatio, height * renderRatio)
if (maxEdge > enforceMaxSideLength) {
  renderRatio *= enforceMaxSideLength / maxEdge
}
```

防止導出超大圖片（例如 10000x10000）耗盡記憶體。預設最長邊 3000px。

#### 繪製到 Canvas 並輸出

```ts
const imgCanvas = getImgCanvas(img, imgLayout, imgAxis.rotate, canvasScale)
// dx, dy 是圖片相對於裁切框的偏移量
let dx = scaledImgAxis.x - scaledCropAxis.x
let dy = scaledImgAxis.y - scaledCropAxis.y

ctx.drawImage(imgCanvas, dx, dy, imgCanvas.width * drawScale, imgCanvas.height * drawScale)

// 輸出 base64 或 blob
if (type === 'blob') {
  canvasToBlob(canvas, outputType, outputSize).then(resolve)
  return
}
const res = canvas.toDataURL(`image/${outputType}`, outputSize)
resolve(res)
```

---

## 二、邊界計算

### 問題描述

當 `centerBox: true` 時，圖片不能被拖出裁切框的範圍。

但圖片可能被旋轉，旋轉後「哪裡算是邊界」就不是簡單的矩形計算了。

### boundaryCalculation：計算圖片的允許範圍

這個函數考慮了圖片旋轉後的邊界：

```ts
// 1. 計算裁切框四個角的座標（旋轉到圖片的座標系）
const cropPoints = [
  { x: cropAxis.x, y: cropAxis.y },                                    // 左上
  { x: cropAxis.x + cropLayout.width, y: cropAxis.y },                 // 右上
  { x: cropAxis.x + cropLayout.width, y: cropAxis.y + cropLayout.height }, // 右下
  { x: cropAxis.x, y: cropAxis.y + cropLayout.height },                // 左下
]
const rotatedCropPoints = cropPoints.map(point => rotatePoint(point, -rotate))

// 2. 計算縮放後的圖片尺寸
const imgWidth = imgLayout.width * scale
const imgHeight = imgLayout.height * scale

// 3. 計算圖片中心允許的範圍（確保圖片完全覆蓋裁切框）
const allowedCenterMinX = Math.max(...rotatedCropPoints.map(p => p.x - halfWidth))
const allowedCenterMaxX = Math.min(...rotatedCropPoints.map(p => p.x + halfWidth))
// ...

// 4. 把當前圖片中心夾在允許範圍內
const targetCenter = rotatePoint(
  { x: clamp(rotatedCenter.x, allowedCenterMinX, allowedCenterMaxX), ... },
  rotate  // 旋轉回螢幕座標系
)
```

**rotatePoint**：把一個點繞原點旋轉指定角度（標準的 2D 旋轉矩陣）：

```ts
const rotatePoint = (point: InterfaceAxis, angle: number): InterfaceAxis => {
  return {
    x: point.x * Math.cos(angle) - point.y * Math.sin(angle),
    y: point.x * Math.sin(angle) + point.y * Math.cos(angle),
  }
}
```

### detectionBoundary：判斷圖片是否越界

```ts
export const detectionBoundary = (
  cropAxis, cropLayout, imgAxis, imgLayout
) => {
  let landscape = ''  // 水平方向越界：'left' | 'right' | ''
  let portrait = ''   // 垂直方向越界：'top' | 'bottom' | ''
  
  const boundary = boundaryCalculation(cropAxis, cropLayout, imgAxis, imgLayout)
  const precision = 0.5  // 允許 0.5px 的浮點誤差
  
  if (imgAxis.x > boundary.left + precision) {
    landscape = 'left'   // 圖片往右偏了，左邊露出空白
  } else if (imgAxis.x < boundary.right - precision) {
    landscape = 'right'  // 圖片往左偏了，右邊露出空白
  }
  
  // ...同樣判斷 portrait
  
  return { landscape, portrait, scale, boundary, imgAxis }
}
```

回傳結果告訴呼叫者：
- `landscape`：水平方向需要往哪邊回彈
- `portrait`：垂直方向需要往哪邊回彈
- `scale`：圖片是否需要放大才能覆蓋裁切框

---

## 三、緩動動畫

### tween.easeInOut：緩動函數

```ts
export const tween = {
  easeInOut: (t: number, b: number, c: number, d: number) => {
    t = (t / d) * 2
    if (t < 1) {
      return (c / 2) * t * t + b         // 前半段：加速
    }
    return (-c / 2) * (--t * (t - 2) - 1) + b  // 後半段：減速
  },
}
```

四個參數：
- `t`：當前時間（幾幀了）
- `b`：起始值
- `c`：變化量（終點 - 起點）
- `d`：總幀數

**easeInOut** = 慢→快→慢（像彈簧效果）。開始和結束都慢，中間快，看起來自然。

---

### setAnimation：執行動畫

```ts
export const setAnimation = (
  from: number,          // 起始值
  to: number,            // 目標值
  duration: number,      // 持續時間（毫秒）
  callback?: (value: number) => void,
) => {
  if (duration <= 0 || from === to) {
    if (callback) callback(to)
    return (): number => 0
  }
  
  let start = 0
  const during = Math.max(1, Math.ceil(duration / 17))  // 幀數（60fps ≈ 17ms/幀）
  let req: number = 0

  const step = () => {
    const value = tween.easeInOut(start, from, to - from, during)
    start++
    if (start <= during) {
      if (callback) callback(value)
      req = requestAnimationFrame(step)  // 下一幀繼續
    } else {
      if (callback) callback(to)  // 動畫結束，確保到達終點
    }
  }
  step()
  return (): number => req
}
```

**requestAnimationFrame**：瀏覽器提供的 API，在下一次螢幕重繪前執行回調。

使用 `requestAnimationFrame` 的好處：
- 動畫和螢幕重新整理率同步（一般是 60fps）
- 頁籤切到背景時，動畫會自動暫停（省電）
- 比 `setInterval` 更精確、更省效能

**使用方式**（在 `vue-cropper.vue` 裡）：

```ts
// 圖片回彈到 x = 50 的位置，100ms 內完成
setAnimation(imgAxis.x, 50, 100, value => {
  setImgAxis({ x: value, y: imgAxis.y })
})
```

---

## 小結

`common.ts` 後段提供了裁剪元件最核心的三個能力：

| 功能 | 函數 | 用途 |
|------|------|------|
| Canvas 操作 | `getImgCanvas`、`getCropImgData` | 把圖片畫進 canvas，輸出裁切結果 |
| 邊界計算 | `boundaryCalculation`、`detectionBoundary` | 判斷圖片是否越界、應回彈到哪 |
| 緩動動畫 | `tween`、`setAnimation` | requestAnimationFrame 驅動的平滑動畫 |

搭配 [07 — 前段](./07-layout-utils.md) 的佈局計算，`common.ts` 就是這個元件的「算術引擎」。

下一篇：[09 — lib/changeImgSize.ts + lib/touch/index.ts（縮放與觸控）](./09-scale-touch.md)
