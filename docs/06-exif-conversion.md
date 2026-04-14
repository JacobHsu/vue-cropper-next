# 06 — lib/exif.ts + lib/conversion.ts（圖片方向校正）

## 為什麼需要方向校正？

用手機拍照時，相機感光元件實際上是**橫置**的。當你豎著拿手機拍，手機不會真正旋轉圖片的像素，而是在圖片的 EXIF 元資料裡記錄「這張照片需要旋轉 90°才是正確方向」。

大多數圖片瀏覽器（如 iOS 相簿、Windows 相片）會自動讀取這個 EXIF 方向並顯示正確的方向。但瀏覽器的 `<img>` 標籤早期不會自動處理，在舊版 Chrome (< 81) 和某些 Safari 版本，圖片會顯示「橫倒」。

`exif.ts` 負責**讀取 EXIF 方向值**，`conversion.ts` 負責**根據這個值在 Canvas 上把圖片轉回正確方向**。

---

## exif.ts 的職責（概要說明）

### 整體流程

```
圖片 URL (string)
       ↓
  getImageData()     ← 把圖片轉成 ArrayBuffer（二進制原始資料）
       ↓
  getOrientation()   ← 從 ArrayBuffer 裡解析 EXIF 標頭，找出方向值
       ↓
  orientation: 1~8   ← 回傳 1（不需旋轉）到 8（需逆時針90°）之一
```

### 支援三種圖片來源

| 來源 | 處理方式 |
|------|---------|
| `data:...`（Base64 內嵌）| 直接解碼 Base64 → ArrayBuffer |
| `blob:...`（Blob URL）| 用 `FileReader` 讀取成 ArrayBuffer |
| `http://...`（遠端 URL）| 用 `XMLHttpRequest` 下載，取得 ArrayBuffer |

### EXIF 方向值的意義

JPEG 圖片的 EXIF 欄位包含一個 Orientation tag，值從 1 到 8：

| 值 | 意思 | 需要的操作 |
|----|------|-----------|
| 1 | 正常（未旋轉）| 不做任何處理 |
| 3 | 旋轉 180° | 旋轉 180° |
| 6 | 順時針旋轉 90°（最常見）| 逆時針旋轉 90° 校正 |
| 8 | 逆時針旋轉 90° | 順時針旋轉 90° 校正 |

值 2、4、5、7 是翻轉+旋轉的組合，相機較少產生，但程式也有處理。

### 何時不需要 EXIF 處理？

現代瀏覽器（Chrome >= 81、Safari >= 13.4）已自動處理 EXIF 方向。`common.ts` 的 `checkOrientationImage()` 函數會偵測瀏覽器版本：

```
Chrome >= 81  → 回傳 -1（不需處理）
Safari >= 13.4 → 回傳 -1（不需處理）
其他          → 回傳原始 orientation 值
```

這樣可以避免「瀏覽器已經校正了，程式又校正一次，結果反而轉錯」。

### exif.ts 的二進制解析（概念說明）

`getOrientation()` 的工作是從 JPEG 的二進制資料裡找出方向值。JPEG 格式有固定的結構：

```
0xFFD8          ← JPEG 開頭標記
0xFFE1 + 長度   ← APP1 區段（EXIF 資料放這裡）
"Exif\0\0"     ← EXIF 識別字
TIFF header    ← 判斷是 Little Endian 還是 Big Endian
IFD entries    ← 每條 12 bytes，tag 0x0112 就是 Orientation
```

程式碼裡的 `dataView.getUint8(offset)` 和 `getUint16()` 就是逐步定位這些區段、最終讀出 Orientation 值的過程。

這個檔案可以**直接複製使用**，不需要修改。如果要替換，npm 上的 `exifr` 是功能更完整的選擇。

---

## conversion.ts 的職責（詳細說明）

`conversion.ts` 使用 Canvas API 把「方向歪掉」的圖片重新繪製成正確方向。

### 核心概念：Canvas 的座標系統

Canvas 的預設座標系統：
- 原點 (0, 0) 在**左上角**
- x 軸向右遞增
- y 軸向下遞增

要旋轉圖片，我們用 `ctx.rotate(angle)` 旋轉**整個座標系統**，再繪製圖片。

---

### Conversion 類別結構

```ts
class Conversion {
  ctx: CanvasRenderingContext2D | null
  img: HTMLImageElement | null
  handle: any

  constructor() {
    this.ctx = null
    this.img = null
    this.handle = {
      1: ..., 2: ..., 3: ..., 4: ...,
      5: ..., 6: ..., 7: ..., 8: ...,
    }
  }
}
```

`handle` 是一個以**方向值為 key** 的物件，每個 key 對應一個「如何校正這個方向的函數」。

---

### render 方法：主入口

```ts
render(img: HTMLImageElement, canvas: HTMLCanvasElement, orientation: number): HTMLCanvasElement {
  this.ctx = canvas.getContext('2d') as CanvasRenderingContext2D
  this.img = img
  
  if (this.handle[orientation]) {
    return this.handle[orientation](canvas)
  }
  
  // 預設：不旋轉，直接繪製
  canvas.width = img.width
  canvas.height = img.height
  this.ctx.drawImage(img, 0, 0, img.width, img.height)
  return canvas
}
```

`render` 接收圖片元素、空白 canvas、方向值，回傳已繪製（且方向校正好）的 canvas。

---

### 方向 1：正常，不需處理

```ts
1: (canvas) => {
  canvas.width = this.img.width
  canvas.height = this.img.height
  // ctx 預設就在 (0,0)，直接 drawImage 就好
  return canvas
}
```

這個最簡單：設定 canvas 尺寸等於圖片尺寸，直接畫。

---

### 方向 3：旋轉 180°

```ts
3: (canvas) => {
  canvas.width = this.img.width
  canvas.height = this.img.height
  this.ctx.translate(this.img.width, this.img.height)
  this.ctx.rotate(Math.PI)   // 旋轉 180°（π 弧度）
  this.ctx.drawImage(this.img, 0, 0)
  return canvas
}
```

**為什麼要先 `translate`？**

`ctx.rotate()` 是以**原點為中心**旋轉。如果不移動原點，旋轉 180° 後圖片會跑到 canvas 的負座標區域（看不見）。

先把原點移到圖片右下角，旋轉 180° 後圖片的「原左上角」就會落在 canvas 左上角，位置正確。

---

### 方向 6：順時針旋轉 90°（最常見，豎拍橫顯）

```ts
6: (canvas) => {
  // 注意：canvas 的寬高要對調
  canvas.width = this.img.height
  canvas.height = this.img.width
  this.ctx.translate(this.img.height, 0)
  this.ctx.rotate(Math.PI / 2)   // 旋轉 90°（π/2 弧度）
  this.ctx.drawImage(this.img, 0, 0)
  return canvas
}
```

這是最常見的情境：用手機**豎拍**，瀏覽器顯示時圖片變**橫的**。

旋轉 90° 後圖片的寬和高對調了，所以 canvas 的尺寸也要對調：

```
原圖：寬 1080, 高 1920
Canvas：寬 1920, 高 1080（對調）
```

---

### 方向 2、4：水平/垂直翻轉

```ts
2: (canvas) => {
  // 水平翻轉（鏡像）
  canvas.width = this.img.width
  canvas.height = this.img.height
  this.ctx.translate(this.img.width, 0)
  this.ctx.scale(-1, 1)   // x 軸鏡像
  this.ctx.drawImage(this.img, 0, 0)
  return canvas
},

4: (canvas) => {
  // 垂直翻轉
  canvas.width = this.img.width
  canvas.height = this.img.height
  this.ctx.translate(0, this.img.height)
  this.ctx.scale(1, -1)   // y 軸鏡像
  this.ctx.drawImage(this.img, 0, 0)
  return canvas
},
```

`ctx.scale(-1, 1)` 把 x 軸方向反轉（水平鏡像）。先 `translate` 是為了讓翻轉後的圖片落在正確位置。

---

## 完整的 lib/conversion.ts

```ts
class Conversion {
  ctx: CanvasRenderingContext2D | null
  img: HTMLImageElement | null
  handle: any

  constructor() {
    this.ctx = null
    this.img = null

    this.handle = {
      1: (canvas: HTMLCanvasElement): HTMLCanvasElement => {
        if (canvas && this.ctx && this.img) {
          canvas.width = this.img.width
          canvas.height = this.img.height
        }
        return canvas
      },
      2: (canvas: HTMLCanvasElement): HTMLCanvasElement => {
        if (canvas && this.ctx && this.img) {
          canvas.width = this.img.width
          canvas.height = this.img.height
          this.ctx.translate(this.img.width, 0)
          this.ctx.scale(-1, 1)
        }
        return canvas
      },
      3: (canvas: HTMLCanvasElement): HTMLCanvasElement => {
        if (canvas && this.ctx && this.img) {
          canvas.width = this.img.width
          canvas.height = this.img.height
          this.ctx.translate(this.img.width, this.img.height)
          this.ctx.rotate(Math.PI)
        }
        return canvas
      },
      4: (canvas: HTMLCanvasElement): HTMLCanvasElement => {
        if (canvas && this.ctx && this.img) {
          canvas.width = this.img.width
          canvas.height = this.img.height
          this.ctx.translate(0, this.img.height)
          this.ctx.scale(1, -1)
        }
        return canvas
      },
      5: (canvas: HTMLCanvasElement): HTMLCanvasElement => {
        if (canvas && this.ctx && this.img) {
          canvas.width = this.img.height
          canvas.height = this.img.width
          this.ctx.rotate(0.5 * Math.PI)
          this.ctx.scale(1, -1)
        }
        return canvas
      },
      6: (canvas: HTMLCanvasElement): HTMLCanvasElement => {
        if (canvas && this.ctx && this.img) {
          canvas.width = this.img.height
          canvas.height = this.img.width
          this.ctx.translate(this.img.height, 0)
          this.ctx.rotate(0.5 * Math.PI)
        }
        return canvas
      },
      7: (canvas: HTMLCanvasElement): HTMLCanvasElement => {
        if (canvas && this.ctx && this.img) {
          canvas.width = this.img.height
          canvas.height = this.img.width
          this.ctx.translate(this.img.height, this.img.width)
          this.ctx.rotate(0.5 * Math.PI)
          this.ctx.scale(-1, 1)
        }
        return canvas
      },
      8: (canvas: HTMLCanvasElement): HTMLCanvasElement => {
        if (canvas && this.ctx && this.img) {
          canvas.width = this.img.height
          canvas.height = this.img.width
          this.ctx.translate(0, this.img.width)
          this.ctx.rotate(-0.5 * Math.PI)
        }
        return canvas
      },
    }
  }

  render(img: HTMLImageElement, canvas: HTMLCanvasElement, orientation: number): HTMLCanvasElement {
    this.ctx = canvas.getContext('2d') as CanvasRenderingContext2D
    this.img = img

    if (this.handle[orientation]) {
      this.handle[orientation](canvas)
      if (this.ctx && this.img) {
        this.ctx.drawImage(this.img, 0, 0)
      }
      return canvas
    }

    canvas.width = img.width
    canvas.height = img.height
    this.ctx.drawImage(img, 0, 0, img.width, img.height)
    return canvas
  }
}

export default Conversion
```

---

## 整體流程回顧

```
載入圖片 URL
     ↓
exif.getData(img)       ← 取得 orientation（1~8 或 -1）
     ↓
checkOrientationImage() ← 現代瀏覽器回傳 -1（不需處理）
     ↓
conversion.render(img, canvas, orientation)
     ↓
已校正方向的 canvas     ← 後續所有操作基於這個 canvas
```

從這個流程可以看出：`exif.ts` 和 `conversion.ts` 只在圖片**第一次載入**時執行一次，之後的拖曳、縮放、旋轉都是在已校正好的 canvas 上操作。

下一篇：[07 — lib/layoutBox.ts + lib/common.ts 前段（佈局計算）](./07-layout-utils.md)
