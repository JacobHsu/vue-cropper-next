# 16 — lib/vue-cropper.vue Part 4：裁切框操作與導出

## 裁切框的視覺原理

在開始讀程式碼之前，先理解裁切框是怎麼「看起來」的。

裁切框（`.cropper-crop-box`）裡面有一張**和主圖一樣的圖片**（`.cropper-view-box img`）。

這張圖片用和主圖**完全相同的 CSS transform**（相同的 scale、rotate），但位移不同：裁切框裡的圖片的偏移 = 主圖偏移 - 裁切框位置。

效果：裁切框的位置好像「開了一個窗口」，透過這個窗口看到原圖的對應部分，且沒有遮罩的暗色效果。

```
主圖（帶遮罩）         裁切框
┌──────────────┐    ┌──────┐
│░░░░░░░░░░░░░░│    │      │  ← 同一張圖，但只顯示這個範圍
│░░░┌──────┐░░│    │  原  │
│░░░│      │░░│    │  圖  │
│░░░│  圖  │░░│    │      │
│░░░│      │░░│    └──────┘
│░░░└──────┘░░│
│░░░░░░░░░░░░░░│
└──────────────┘
```

---

## 渲染裁切框：renderCrop

```ts
const renderCrop = (axis?: InterfaceAxis): void => {
  const { width, height } = LayoutContainer.wrapLayout
  let cropW = cropLayoutStyle.value.width
  let cropH = cropLayoutStyle.value.height

  // 裁切框不能超過容器
  if (width > 0) cropW = Math.min(cropW, width)
  if (height > 0) cropH = Math.min(cropH, height)

  // 預設裁切框居中
  const defaultAxis: InterfaceAxis = {
    x: (width - cropW) / 2,
    y: (height - cropH) / 2,
  }

  if (axis) {
    checkedCrop(axis)  // 有指定位置：驗證並使用
  } else {
    checkedCrop(defaultAxis)  // 沒有指定：居中
  }
}
```

**為什麼裁切框要受容器大小限制？**

使用者可能設定 `cropLayout: { width: 500, height: 500 }` 但容器只有 300x300。這時把裁切框尺寸限制在容器內，避免裁切框超出容器邊界。

---

## 裁切框位置校驗：checkedCrop

```ts
const checkedCrop = (axis: InterfaceAxis) => {
  const maxLeft = 0                                              // 最左：緊貼容器左邊
  const maxTop = 0                                               // 最上：緊貼容器頂部
  const cropWidth = effectiveCropLayoutStyle.value.width
  const cropHeight = effectiveCropLayoutStyle.value.height
  const maxRight = LayoutContainer.wrapLayout.width - cropWidth   // 最右：裁切框右邊對齊容器右邊
  const maxBottom = LayoutContainer.wrapLayout.height - cropHeight // 最下：裁切框底部對齊容器底部

  // 夾在允許範圍內（不能超出容器）
  if (axis.x < maxLeft) axis.x = maxLeft
  if (axis.y < maxTop) axis.y = maxTop
  if (axis.x > maxRight) axis.x = maxRight
  if (axis.y > maxBottom) axis.y = maxBottom

  LayoutContainer.cropAxis = axis
  cropping.value = true    // 確保裁切框是顯示狀態
  queueRealTimeEmit()
}
```

裁切框的位置限制很簡單：四個邊都不能超出容器。

---

## 移動裁切框：moveCrop

```ts
const moveCrop = (message: InterfaceMessageEvent) => {
  if (isImgTouchScale.value) return  // 雙指縮放時，禁止移動裁切框

  if (message.change) {
    const axis = {
      x: message.change.x + LayoutContainer.cropAxis.x,
      y: message.change.y + LayoutContainer.cropAxis.y,
    }
    checkedCrop(axis)  // 計算新位置並校驗
  }
}
```

和 `moveImg` 邏輯相同：新座標 = 舊座標 + 移動量。

差別在於沒有「阻力」效果——裁切框直接被邊界擋住，不能超出容器。

---

## 清除裁切框：clearCrop

```ts
const clearCrop = () => {
  LayoutContainer.cropAxis = { x: 0, y: 0 }
  cropping.value = false  // 隱藏裁切框
}
```

當使用者不需要裁切框時（如使用 `original` 模式），可以呼叫這個方法。

---

## 裁切框的 CSS 樣式計算

### getCropBoxStyle：外層裁切框

```ts
const getCropBoxStyle = (): InterfaceTransformStyle => {
  const style = {
    width: `${effectiveCropLayoutStyle.value.width}px`,
    height: `${effectiveCropLayoutStyle.value.height}px`,
    transform: `translate3d(${LayoutContainer.cropAxis.x}px, ${LayoutContainer.cropAxis.y}px, 0)`,
  }
  LayoutContainer.cropExhibitionStyle.div = style
  return style
}
```

裁切框的移動用 `translate3d`（讓 GPU 處理），位移值就是 `cropAxis.x` 和 `cropAxis.y`。

### getCropImgStyle：裁切框內部的圖片

```ts
const getCropImgStyle = (): InterfaceTransformStyle => {
  const scale = LayoutContainer.imgAxis.scale
  
  // 裁切框內的圖片偏移 = 主圖偏移 - 裁切框位置 + 縮放補償
  const x =
    ((LayoutContainer.imgLayout.width * (scale - 1)) / 2 +
      (LayoutContainer.imgAxis.x - LayoutContainer.cropAxis.x)) / scale
  const y =
    ((LayoutContainer.imgLayout.height * (scale - 1)) / 2 +
      (LayoutContainer.imgAxis.y - LayoutContainer.cropAxis.y)) / scale

  const style = {
    width: `${LayoutContainer.imgLayout.width}px`,
    height: `${LayoutContainer.imgLayout.height}px`,
    transform: `scale(${scale}, ${scale}) translate3d(${x}px, ${y}px, 0) rotateZ(${LayoutContainer.imgAxis.rotate}deg)`,
  }
  LayoutContainer.cropExhibitionStyle.img = style
  return style
}
```

**關鍵公式**：`x = (縮放補償 + 圖片座標 - 裁切框座標) / scale`

這個公式讓裁切框內的圖片和主圖的視覺位置完全對齊——無論圖片怎麼移動、縮放，裁切框裡看到的始終是「裁切框對應範圍的圖片」。

---

## 導出裁切結果

### getCropData：取得裁切圖片（base64 或 blob）

```ts
const getCropData = (type: 'base64' | 'blob' = 'base64') => {
  const obj = {
    type,
    outputType: outputType.value,
    outputSize: outputSize.value,
    full: full.value,
    original: original.value,
    maxSideLength: maxSideLength.value,
    url: imgs.value,
    imgAxis: { ...LayoutContainer.imgAxis },        // 圖片的位置/縮放/旋轉
    imgLayout: { ...LayoutContainer.imgLayout },    // 圖片的原始尺寸
    cropLayout: { ...effectiveCropLayoutStyle.value }, // 裁切框的尺寸
    cropAxis: { ...LayoutContainer.cropAxis },      // 裁切框的位置
    cropping: cropping.value,
  }
  return getCropImgData(obj)  // common.ts 裡的函數
}
```

**為什麼所有狀態都要用展開（`{...}`）複製？**

如果直接傳 `LayoutContainer.imgAxis`（物件引用），`getCropImgData` 可能還在非同步執行時，`LayoutContainer.imgAxis` 已經被使用者的操作改變了。

複製一份快照，確保導出的是「呼叫時的狀態」，不受後續操作影響。

### getCropBlob：便捷方法（Blob 格式）

```ts
const getCropBlob = () => {
  return getCropImgData({
    type: 'blob',
    ...  // 和 getCropData 相同
  }) as Promise<Blob>
}
```

**使用方式（在父元件裡）**：

```html
<VueCropper ref="cropperRef" :img="imgUrl" />
<button @click="download">下載</button>

<script setup>
const cropperRef = ref()

const download = async () => {
  // 取得 base64
  const base64 = await cropperRef.value.getCropData()

  // 或取得 Blob
  const blob = await cropperRef.value.getCropBlob()
  const link = document.createElement('a')
  link.href = URL.createObjectURL(blob)
  link.download = 'crop.png'
  link.click()
}
</script>
```

---

## 拖曳上傳

```ts
const drop = (e: DragEvent) => {
  e.preventDefault()
  const dataTransfer = e.dataTransfer as DataTransfer
  isDrag.value = false  // 隱藏拖曳提示
  loadFile(dataTransfer.files[0]).then(res => {
    if (res) imgUploadEmit(res)
  })
}

const dragover = (e: Event) => {
  e.preventDefault()
  isDrag.value = true   // 顯示拖曳提示
}

const dragend = (e: Event) => {
  e.preventDefault()
  isDrag.value = false
}
```

使用者可以直接把圖片檔案拖曳進裁剪元件。

**`isDrag`**：`true` 時顯示「拖動圖片到此」的提示區域（`.drag` slot）。

**父元件的完整使用方式**：

```html
<VueCropper
  :img="imgUrl"
  @img-upload="url => { imgUrl = url }"
/>
```

`img-upload` 事件回傳拖曳進來的圖片 URL，父元件更新 `imgUrl` prop，元件重新載入圖片。

---

## defineExpose：開放給父元件的方法

```ts
defineExpose({
  getCropData,   // 取得裁切結果（base64）
  getCropBlob,   // 取得裁切結果（Blob）
  rotateLeft,    // 向左旋轉 90°
  rotateRight,   // 向右旋轉 90°
  rotateClear,   // 重置旋轉（回到 0°）
})
```

這五個方法是外部可以「呼叫」元件的完整 API。

---

## 小結

裁切框的三大職責：

| 職責 | 函數 | 說明 |
|------|------|------|
| 視覺呈現 | `getCropBoxStyle`、`getCropImgStyle` | 計算 CSS，讓裁切框看起來正確 |
| 互動操作 | `moveCrop`、`renderCrop`、`checkedCrop` | 拖曳、初始化、邊界限制 |
| 導出結果 | `getCropData`、`getCropBlob` | 把當前狀態交給 `common.ts` 計算出圖片 |

下一篇：[17 — lib/vue-cropper.vue Part 5：Template 結構與生命週期](./17-main-comp-5-template.md)
