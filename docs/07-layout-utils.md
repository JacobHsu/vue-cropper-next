# 07 — lib/layoutBox.ts + lib/common.ts 前段（佈局計算）

## 這篇解決什麼問題？

圖片載入後需要決定：**圖片要以多大的比例顯示在容器裡？**

- 一張 4000x3000 的照片，放進 300x300 的容器，要縮多少？
- 使用者可以選擇 `cover`（填滿容器）或 `contain`（完整顯示）

`layoutBox.ts` 負責回答這個問題，回傳一個**縮放比例（scale）**。

`common.ts` 前段則提供：圖片載入、EXIF 方向校正包裝、CSS transform 計算等工具函數。

---

## lib/layoutBox.ts：計算初始縮放比例

### 三種佈局模式

```
contain: 圖片縮到完整顯示在容器內（不裁切）
cover:   圖片放大到填滿容器（可能超出邊界）
default: 自訂（支援 px、%、original 等格式）
```

這三種模式對應 CSS 的 `background-size` 概念，只是這裡是用 Canvas 計算，而不是 CSS。

---

### 主入口函數

```ts
const layout = (
  imgStyle: InterfaceLayoutStyle,    // 圖片原始寬高
  layoutStyle: InterfaceLayoutStyle, // 容器寬高
  mode: keyof InterfaceModeHandle,   // 'contain' | 'cover' | 'default'
): number => {
```

回傳值是一個純數字，表示**縮放比例**：
- `1.0` = 圖片以原始大小顯示
- `0.5` = 圖片縮小到 50%
- `2.0` = 圖片放大到 200%

函數先找到對應的處理模式，再呼叫它：

```ts
  let handleType: keyof InterfaceModeHandle = 'default'
  for (const key of Object.keys(modeHandle)) {
    if (key === mode) {
      handleType = mode
      break
    }
  }
  return modeHandle[handleType](imgStyle, layoutStyle, mode)
}
```

---

### contain 模式：完整顯示

```ts
contain: (imgStyle, layoutStyle): number => {
  let scale = 1
  const { width, height } = imgStyle
  const wrapWidth = layoutStyle.width
  const wrapHeight = layoutStyle.height

  if (width > wrapWidth) {
    scale = wrapWidth / width       // 先按寬度縮放
  }
  if (height * scale > wrapHeight) {
    scale = wrapHeight / height     // 如果高度還是超出，再按高度縮放
  }
  return scale
},
```

**例子**：圖片 800x400，容器 300x300

1. 寬度超出：`scale = 300 / 800 = 0.375`
2. 縮放後高度：`400 * 0.375 = 150` → 不超出 300，不需再縮
3. 結果：scale = 0.375，圖片顯示為 300x150，置中在容器裡

---

### cover 模式：填滿容器

```ts
cover: (imgStyle, layoutStyle): number => {
  let scale = 1
  const { width, height } = imgStyle
  const wrapWidth = layoutStyle.width
  const wrapHeight = layoutStyle.height

  scale = wrapWidth / width                // 先按寬度放大到填滿
  const curHeight = height * scale
  if (curHeight < wrapHeight) {
    scale = wrapHeight / height            // 如果高度不夠，再按高度放大
  }
  return scale
},
```

**例子**：圖片 800x400，容器 300x300

1. 按寬度：`scale = 300 / 800 = 0.375`，高度變 `400 * 0.375 = 150`
2. 高度 150 < 容器高度 300 → 不夠高
3. 改按高度：`scale = 300 / 400 = 0.75`
4. 結果：scale = 0.75，圖片顯示為 600x300，超出容器寬度被裁掉

---

### default 模式：自訂尺寸

支援多種格式：

```
'original'     → scale = 1（原始大小）
'100px'        → 寬度強制 100px
'50%'          → 寬度為容器的 50%
'auto 200px'   → 高度強制 200px（寬度自動）
'auto 50%'     → 高度為容器的 50%（寬度自動）
```

```ts
default: (imgStyle, layoutStyle, mode): number => {
  let scale = 1
  if (mode === 'original') {
    return scale
  }
  let { width, height } = { ...imgStyle }
  const wrapWidth = layoutStyle.width
  const wrapHeight = layoutStyle.height
  const arr = mode.split(' ')  // 分割成 ['100px'] 或 ['auto', '200px']
  
  try {
    let str = arr[0]
    if (str.search('px') !== -1) {
      width = parseFloat(str.replace('px', ''))
      scale = width / imgStyle.width
    }
    if (str.search('%') !== -1) {
      width = (parseFloat(str.replace('%', '')) / 100) * wrapWidth
      scale = width / imgStyle.width
    }
    if (arr.length === 2 && str === 'auto') {
      let str2 = arr[1]
      if (str2.search('px') !== -1) {
        height = parseFloat(str2.replace('px', ''))
        scale = height / imgStyle.height
      }
      if (str2.search('%') !== -1) {
        height = (parseFloat(str2.replace('%', '')) / 100) * wrapHeight
        scale = height / imgStyle.height
      }
    }
  } catch (e) {
    console.error(e)
    scale = 1
  }
  return scale
},
```

---

## lib/common.ts 前段：工具函數集合

### loadImg：Promise 包裝圖片載入

```ts
export const loadImg = async (url: string): Promise<HTMLImageElement> => {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.onload = () => resolve(img)
    img.onerror = reject
    img.src = url
    if (url.substr(0, 4) !== 'data') {
      img.crossOrigin = ''
    }
  })
}
```

**為什麼要包裝成 Promise？**

`Image` 的圖片載入是非同步的，但它只支援 `onload` callback。包裝成 Promise 後，就能把「等圖片載入完成再往下做」寫成 `await loadImg(url)`；這樣後續程式可以直接寫在下一行，不必全部包進 `onload = () => { ... }` 裡，流程會更直觀。

**`crossOrigin = ''`**：跨域圖片需要設定這個屬性，才能讓 Canvas 讀取圖片像素。如果不設，`canvas.toDataURL()` 會因安全限制拋出錯誤。

Base64 圖片（`data:image/...`）不需要這個設定，所以先判斷 URL 開頭。

---

### getExif + resetImg：包裝方向校正流程

```ts
export const getExif = (img: HTMLImageElement): Promise<any> => {
  return exif.getData(img)
}

export const resetImg = (
  img: HTMLImageElement,
  canvas: HTMLCanvasElement | null,
  orientation: number,
): HTMLCanvasElement | null => {
  if (!canvas) return canvas
  return conversion.render(img, canvas, orientation)
}
```

這兩個函數只是對 `exif.ts` 和 `conversion.ts` 的薄薄包裝，讓 `vue-cropper.vue` 不需要直接引用 `exif` 和 `conversion`，保持依賴關係清晰。

---

### checkOrientationImage：判斷是否需要 EXIF 校正

```ts
export const checkOrientationImage = (orientation: number) => {
  // Chrome >= 81 自動處理 EXIF
  if (Number(getVersion('chrome')[0]) >= 81) {
    return -1
  }
  // Safari >= 13.4 自動處理 EXIF
  if (Number(getVersion('safari')[0]) >= 605) {
    const safariVersion = getVersion('version')
    if (Number(safariVersion[0]) > 13 && Number(safariVersion[1]) > 1) {
      return -1
    }
  }
  // iOS >= 13.4 自動處理 EXIF
  const isIos = navigator.userAgent.toLowerCase()
    .match(/cpu iphone os (.*?) like mac os/)
  if (isIos) {
    const version = isIos[1].split('_').map(Number)
    if (version[0] > 13 || (version[0] >= 13 && version[1] >= 4)) {
      return -1
    }
  }
  return orientation
}
```

回傳 `-1` 表示「不需要 EXIF 校正」。

**getVersion 輔助函數**：從 `navigator.userAgent` 字串裡提取指定瀏覽器的版本號。

---

### createImgStyle：計算初始 scale

```ts
export const createImgStyle = (
  imgStyle: InterfaceLayoutStyle,
  layoutStyle: InterfaceLayoutStyle,
  mode: keyof InterfaceModeHandle,
): number => {
  return layout(imgStyle, layoutStyle, mode)
}
```

這個函數只是把 `layoutBox` 函數重新導出，讓使用者統一從 `common.ts` 引用。

---

### translateStyle：把 scale + 座標轉成 CSS transform 字串

這是 common.ts 前段最重要的函數，負責把「數學計算結果」轉成「CSS 可以使用的字串」。

```ts
export const translateStyle = (
  style: InterfaceRenderImgLayout,
  axis?: InterfaceAxis,
): any => {
  const { scale, imgStyle, layoutStyle, rotate } = style
  
  // 縮放後的圖片尺寸
  const curStyle = {
    width: scale * imgStyle.width,
    height: scale * imgStyle.height,
  }
  
  // 預設居中：x = (容器寬 - 圖片寬) / 2
  let x = (layoutStyle.width - curStyle.width) / 2
  let y = (layoutStyle.height - curStyle.height) / 2
  
  if (axis) {
    x = axis.x
    y = axis.y
  }
```

**為什麼需要 `left` 和 `top` 的額外計算？**

CSS 的 `transform: scale()` 是以元素**中心為原點**縮放，而我們想要的是以**左上角為原點**控制位置。

如果圖片縮放後寬度是 `scale * width`，那麼縮放導致的「中心偏移」是 `(scale * width - width) / 2 = (curStyle.width - imgStyle.width) / 2`。

要讓圖片的左上角落在 `(x, y)` 位置，translate 的值需要補償這個偏移：

```ts
const left = ((curStyle.width - imgStyle.width) / 2 + x) / scale
const top = ((curStyle.height - imgStyle.height) / 2 + y) / scale
```

**為什麼除以 scale？**

`translate3d` 的值在 `scale()` 的座標系下，所以實際移動距離 = 設定值 × scale。  
要達到「在原始座標系移動 x px」，需要設定 `x / scale`。

```ts
  return {
    imgExhibitionStyle: {
      width: `${imgStyle.width}px`,    // 圖片原始尺寸（不含縮放）
      height: `${imgStyle.height}px`,
      transform: `scale(${scale}, ${scale}) translate3d(${left}px, ${top}px, 0px) rotateZ(${rotate}deg)`,
    },
    imgAxis: {
      x,     // 保存「圖片左上角」的邏輯座標
      y,
      scale,
      rotate,
    },
  }
}
```

**為什麼 `width` / `height` 用原始尺寸，縮放用 CSS transform？**

把縮放交給 GPU 處理的 CSS transform 比 Canvas 重繪更高效，而且 `scale()` + `translate3d()` 可以觸發硬體加速（GPU compositing）。

> **瀏覽器渲染的三個步驟**
>
> 瀏覽器把「資料 → 畫面」分成三個工作：
>
> ```
> Layout（排版）    → CPU：計算每個元素的位置和大小
> Paint（繪圖）     → CPU：把元素畫成像素
> Composite（合成） → GPU：把各圖層疊在一起輸出到螢幕
> ```
>
> 當你改變 `left`、`top`、`width`、`height` 時，瀏覽器需要重新計算「這個元素移動後，旁邊的元素有沒有被擠到」，觸發 **Layout**，整個 DOM 樹都要重算——這個過程叫做 **reflow**，很耗效能。
>
> 而 `translate3d` 和 `scale` 屬於只影響 **Composite** 的屬性：瀏覽器把元素提升為一個獨立的 GPU 圖層，移動時只是告訴 GPU「把這個圖層往右移 Xpx」，完全不觸發 Layout，CPU 不需要參與。
>
> 裁切框拖曳、圖片縮放這類操作每秒需要更新 60 次，用 `translate3d` 才能保持流暢，不掉幀。
>
> **本專案的對應程式碼：**
> - 圖片樣式（縮放 + 移動 + 旋轉）：`lib/common.ts` → `translateStyle()`，回傳的 `transform` 字串格式為 `scale() translate3d() rotateZ()`
> - 裁切框定位：`lib/vue-cropper.vue` → `getCropBoxStyle()`，`transform: translate3d(x, y, 0)`
> - 裁切框內圖片（跟著主圖同步）：`lib/vue-cropper.vue` → `getCropImgStyle()`，同樣的 `scale() translate3d() rotateZ()` 格式

---

## 小結

| 函數 | 位置 | 職責 |
|------|------|------|
| `layout()` | `layoutBox.ts` | 計算初始縮放比例（contain/cover/custom）|
| `createImgStyle()` | `common.ts` | 包裝 layout，對外介面 |
| `translateStyle()` | `common.ts` | 把 scale + 座標 → CSS transform 字串 |
| `loadImg()` | `common.ts` | Promise 包裝圖片載入 |
| `checkOrientationImage()` | `common.ts` | 判斷是否需要 EXIF 校正 |

下一篇：[08 — lib/common.ts 後段（Canvas 操作、邊界計算、動畫）](./08-common-canvas.md)
