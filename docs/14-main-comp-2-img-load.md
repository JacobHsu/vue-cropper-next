# 14 — lib/vue-cropper.vue Part 2：圖片加載流程

## 圖片加載的完整流程

```
Props.img 改變
       ↓
  watch(img) 觸發
       ↓
  checkedImg(url)         ← 主入口：EXIF 校正 → 濾鏡 → 建立 Blob URL
       ↓
  renderFilter()          ← 套用濾鏡（如果有的話）
       ↓
  createImg()             ← canvas.toBlob → 建立 Blob URL
       ↓
  renderImgLayout(url)    ← 計算圖片的初始縮放比例和容器尺寸
       ↓
  translateStyle(...)     ← 計算 CSS transform
       ↓
  imgs.value = url        ← 觸發 watch(imgs)
       ↓
  bindMoveImg()           ← 綁定拖曳事件
  bindMoveCrop()          ← 綁定裁切框事件
  renderCrop()            ← 渲染裁切框到容器中央
```

---

## checkedImg：主入口

```ts
const checkedImg = async (url: string) => {
  imgLoading.value = true  // 顯示 loading
  imgs.value = ''          // 清空目前圖片（避免殘留）
  canvas = null

  let img: HTMLImageElement
  try {
    img = await loadImg(url)
    imgLoadEmit({ type: 'success', message: '图片加载成功' })
  } catch (error) {
    imgLoadEmit({ type: 'error', message: `图片加载失败${error}` })
    imgLoading.value = false
    return false
  }
```

**為什麼先把 `imgs.value` 清空？**

如果使用者切換圖片，舊圖片會先消失（`imgs.value = ''`），再顯示新圖片。這樣可以避免「新舊圖片閃一下」的視覺問題。

**try-catch 的意義**：
- 網路圖片 404 會觸發 `onerror`
- 攔截到錯誤 → 發送 `img-load` 事件（type: 'error'）讓父元件知道
- 不讓錯誤傳播到外面造成整個頁面崩潰

```ts
  // 取得 EXIF 方向資訊
  let result = { orientation: -1 }
  try {
    result = await getExif(img)
  } catch (error) {
    result.orientation = 1  // 取不到 EXIF，假設方向正確（1 = 不需旋轉）
  }

  let orientation = result.orientation || -1
  orientation = checkOrientationImage(orientation) as number
```

**為什麼 `getExif` 也要 try-catch？**

EXIF 解析可能失敗：
- PNG/GIF/WebP 圖片沒有 EXIF 資料
- 某些 JPEG 圖片的 EXIF 格式不標準

失敗時預設 `orientation = 1`（正常方向），繼續往下執行，不讓 EXIF 問題阻斷整個加載流程。

```ts
  // 在空白 canvas 上繪製方向校正後的圖片
  let newCanvas: HTMLCanvasElement = document.createElement('canvas')
  try {
    newCanvas = await resetImg(img, newCanvas, orientation) ?? newCanvas
  } catch (error) {
    console.error(error)
  }
  canvas = newCanvas
  renderFilter()
}
```

**`?? newCanvas`**：`??` 是空值合併運算子（Nullish Coalescing）。

`resetImg` 可能回傳 `null`（當 canvas 參數是 null 時），這時用 `?? newCanvas` 確保 `canvas` 不是 null。

---

## renderFilter：套用濾鏡

```ts
const renderFilter = () => {
  if (filter.value) {
    let newCanvas = canvas as HTMLCanvasElement
    newCanvas = filter.value(newCanvas) ?? newCanvas
    canvas = newCanvas
  }
  createImg()
}
```

如果使用者傳入了 `filter` prop（如 `grayscale`），就在 canvas 上套用濾鏡。

濾鏡函數的簽名是 `(canvas) => canvas`，所以可以直接呼叫並用回傳值更新 canvas。

**為什麼濾鏡在這裡套用，而不是在 `getCropImgData`（導出）時套用？**

在載入時套用：讓使用者**看到**加上濾鏡後的效果，所見即所得。

如果在導出時套用，使用者操作圖片時看到的是無濾鏡版本，導出才有濾鏡，體驗不一致。

---

## createImg：canvas → Blob URL

```ts
const createImg = () => {
  if (!canvas) return

  try {
    canvas.toBlob(
      async blob => {
        if (blob) {
          URL.revokeObjectURL(imgs.value)  // 釋放舊的 Blob URL
          const url = URL.createObjectURL(blob)  // 建立新的 Blob URL

          let scale = 1
          try {
            scale = await renderImgLayout(url)
          } catch (e) {
            console.error(e)
          }

          const style = translateStyle({
            scale,
            imgStyle: { ...LayoutContainer.imgLayout },
            layoutStyle: { ...LayoutContainer.wrapLayout },
            rotate: defaultRotate.value,
          })
          LayoutContainer.imgExhibitionStyle = style.imgExhibitionStyle
          LayoutContainer.imgAxis = style.imgAxis
          imgs.value = url   // ← 這行觸發 watch(imgs)！
          
          if (cropping.value) renderCrop()
          reboundImg()
          queueRealTimeEmit()
          imgLoading.value = false
        } else {
          imgs.value = ''
          imgLoading.value = false
        }
      },
      `image/${outputType.value}`,
      1,  // 最高品質（此時不用壓縮，壓縮在導出時才做）
    )
  } catch (e) {
    console.error(e)
    imgLoading.value = false
  }
}
```

**為什麼要把 canvas 轉成 Blob URL？**

canvas 本身不能直接用在 `<img>` 標籤。流程：

```
canvas（像素資料）→ toBlob() → Blob（二進制資料）→ createObjectURL() → Blob URL（字串）
```

Blob URL 是一個臨時 URL（格式 `blob:http://localhost:5173/xxxxxxx`），指向記憶體裡的 Blob 物件。

**`URL.revokeObjectURL(imgs.value)`**：

Blob URL 佔用記憶體，不再使用時必須手動釋放。每次建立新 URL 前，先釋放舊的。這是避免記憶體洩漏（memory leak）的好習慣。

---

## renderImgLayout：計算初始佈局

```ts
const renderImgLayout = async (url: string): Promise<number> => {
  let img: HTMLImageElement
  try {
    img = await loadImg(url)
  } catch (error) {
    imgLoadEmit({ type: 'error', message: `图片加载失败${error}` })
    imgLoading.value = false
    return 1  // 失敗時預設 scale = 1
  }

  const wrapper = { width: 0, height: 0 }
  if (cropperRef.value) {
    wrapper.width = Number(
      (window.getComputedStyle(cropperRef.value).width || '').replace('px', ''),
    )
    wrapper.height = Number(
      (window.getComputedStyle(cropperRef.value).height || '').replace('px', ''),
    )
  }

  LayoutContainer.imgLayout = { width: img.width, height: img.height }
  LayoutContainer.wrapLayout = { ...wrapper }
  return createImgStyle({ ...LayoutContainer.imgLayout }, wrapper, mode.value)
}
```

**為什麼要用 `window.getComputedStyle()` 讀取容器尺寸，而不是直接用 `props.wrapper.width`？**

`props.wrapper.width` 可能是 `'50%'`（百分比字串），這個相對值需要知道父元素的實際尺寸才能計算出像素值。

`getComputedStyle()` 回傳瀏覽器計算後的**實際像素值**（如 `'300px'`），這才是我們計算佈局需要的值。

---

## watch：監聽 Props 變化

```ts
// 圖片 URL 改變 → 重新載入
watch(img, (val) => {
  if (val && val !== imgs.value) {
    checkedImg(val)
  }
})

// imgs（Blob URL）更新後 → 重新綁定事件
watch(imgs, (val) => {
  if (val) {
    nextTick(() => { bindMoveImg() })
    if (cropping.value && shouldShowCropBox.value) {
      nextTick(() => { bindMoveCrop() })
    }
  }
})

// 濾鏡改變 → 重新載入（因為濾鏡在載入時套用）
watch(filter, () => {
  imgLoading.value = true
  checkedImg(img.value)
})

// 佈局模式改變 → 重新載入（初始縮放比例會改變）
watch(mode, () => {
  imgLoading.value = true
  checkedImg(img.value)
})

// 預設旋轉角度改變 → 立刻旋轉
watch(defaultRotate, (val) => {
  setRotate(val)
})
```

**`nextTick()` 的必要性**：

當 `imgs.value` 改變後，Vue 會排程 DOM 更新（非同步）。但 `bindMoveImg` 需要讀取 `cropperImg.value`（DOM 元素）。如果不等 DOM 更新，讀到的可能是 `undefined`（元素還沒插入）。

`nextTick` 確保等 DOM 更新完再執行。

---

## 容器尺寸改變的處理

```ts
watch(
  wrapperStyle,
  async () => {
    await nextTick()
    updateWrapLayoutFromDom()  // 重新讀取容器實際尺寸
    if (!imgs.value) return
    if (cropping.value) {
      renderCrop({ ...LayoutContainer.cropAxis })  // 維持裁切框位置
    }
    setImgAxis({ x: LayoutContainer.imgAxis.x, y: LayoutContainer.imgAxis.y })
    reboundImg()
  },
  { deep: true, flush: 'post' },
)
```

使用者可能動態改變容器大小（如響應式佈局），這時要重新計算圖片和裁切框的位置。

`flush: 'post'` 確保在 DOM 更新後才執行（因為要讀取 DOM 的計算樣式）。

---

## 即時預覽：getRealTimePreview

```ts
const getRealTimePreview = (): InterfaceRealTimePreview | null => {
  if (!imgs.value || !cropping.value) return null

  const scale = LayoutContainer.imgAxis.scale
  // 計算圖片在裁切框內的偏移
  const transformX =
    ((LayoutContainer.imgLayout.width * (scale - 1)) / 2 +
      (LayoutContainer.imgAxis.x - LayoutContainer.cropAxis.x)) / scale
  const transformY =
    ((LayoutContainer.imgLayout.height * (scale - 1)) / 2 +
      (LayoutContainer.imgAxis.y - LayoutContainer.cropAxis.y)) / scale

  const transform = `scale(${scale}, ${scale}) translate3d(${transformX}px, ${transformY}px, 0) rotateZ(${LayoutContainer.imgAxis.rotate}deg)`
  const width = effectiveCropLayoutStyle.value.width
  const height = effectiveCropLayoutStyle.value.height

  return {
    w: width,
    h: height,
    url: imgs.value,
    img: { width: `...`, height: `...`, transform },
    html: `<div class="show-preview" style="...">...</div>`,  // 可直接塞進 innerHTML
  }
}
```

這個函數計算「如果在其他地方顯示這個裁切框的預覽，需要的樣式資料」。

父元件監聽 `real-time` 事件就能即時更新預覽畫面：

```html
<VueCropper @real-time="updatePreview" />

<div class="preview" :style="{ width: preview.w + 'px', height: preview.h + 'px' }">
  <img :src="preview.url" :style="preview.img" />
</div>
```

---

## queueRealTimeEmit：防抖的即時預覽

```ts
let realTimeFrame = 0

const queueRealTimeEmit = () => {
  if (realTimeFrame) {
    cancelAnimationFrame(realTimeFrame)  // 取消上一次還沒執行的預覽
  }
  realTimeFrame = requestAnimationFrame(() => {
    realTimeFrame = 0
    emitRealTime()
  })
}
```

每次操作（拖曳、縮放、旋轉）都會呼叫這個函數。

如果每次移動都立刻發送事件，一秒內可能發送 60 次（60fps），效能差且沒必要。

用 `requestAnimationFrame` 把多次呼叫合併成「每幀最多一次」：只有當瀏覽器真正要重繪時才發送。

**`cancelAnimationFrame`**：如果上一幀的預覽請求還沒執行（例如在 1ms 內連續移動了 10 次），直接取消，用最新的狀態發送一次。

---

## 小結

圖片加載流程的五個關鍵步驟：

| 步驟 | 函數 | 說明 |
|------|------|------|
| 1 | `checkedImg` | URL → Image → EXIF 校正 → canvas |
| 2 | `renderFilter` | 套用濾鏡（如果有）|
| 3 | `createImg` | canvas → Blob → Blob URL |
| 4 | `renderImgLayout` | 計算初始 scale |
| 5 | `translateStyle` | scale + 座標 → CSS transform |

下一篇：[15 — lib/vue-cropper.vue Part 3：拖曳、縮放、回彈](./15-main-comp-3-move.md)
