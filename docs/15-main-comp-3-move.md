# 15 — lib/vue-cropper.vue Part 3：拖曳、縮放、回彈

## 這篇的範圍

這篇說明圖片的**互動操作**：

- 圖片拖曳移動
- 滑鼠滾輪縮放
- 觸控雙指縮放
- 放開後的邊界回彈
- 旋轉

---

## 事件綁定架構

前面建立了 `TouchEvent` 類別（`touch/index.ts`），它把滑鼠和觸控統一成語意事件。

在 `vue-cropper.vue` 裡，圖片和裁切框各有一個 `TouchEvent` 實例：

```
cropImg = new TouchEvent(cropperImg)  ← 圖片的拖曳感應層
cropBox = new TouchEvent(cropperBox)  ← 裁切框的點擊層
```

**注意**：圖片和裁切框不能同時綁定 `down-to-move`。當裁切框出現時，拖曳圖片的功能實際上是**透過裁切框**來移動的：

```ts
const bindMoveCrop = (): void => {
  unbindMoveCrop()
  const domBox = cropperBox.value
  if (!domBox) return
  cropBox = new TouchEvent(domBox)
  cropBox.on('down-to-move', moveCrop)      // 拖曳裁切框
  cropBox.on('down-to-scale', moveScale)    // 雙指縮放
  cropBox.on('up', reboundImg)
  cropImg = null  // ← 同時清除圖片的拖曳（避免衝突）
}
```

---

## 圖片移動

### moveImg：處理移動量

```ts
const moveImg = (message: InterfaceMessageEvent) => {
  if (message.change) {
    const axis = {
      x: message.change.x + LayoutContainer.imgAxis.x,
      y: message.change.y + LayoutContainer.imgAxis.y,
    }
```

`message.change` 是這幀和上幀的**座標差值**（不是絕對座標）。

新座標 = 舊座標 + 移動量。

**centerBox 的阻力邏輯**：

```ts
    if (centerBox.value) {
      const crossing = detectionBoundary(
        { ...LayoutContainer.cropAxis },
        { ...effectiveCropLayoutStyle.value },
        { ...LayoutContainer.imgAxis },
        { ...LayoutContainer.imgLayout },
      )

      if (crossing.landscape !== '' || crossing.portrait !== '') {
        // 圖片已越界：施加阻力（只移動 RESISTANCE 倍的距離）
        axis.x = LayoutContainer.imgAxis.x + message.change.x * RESISTANCE
        axis.y = LayoutContainer.imgAxis.y + message.change.y * RESISTANCE
      }
    }
    setImgAxis(axis)
  }
}
```

**RESISTANCE = 0.2**：越界後每移動 100px，圖片只移動 20px。

這個「橡皮筋感」告訴使用者「你已經超出了」，放開後圖片會自動彈回。

---

### setImgAxis：更新圖片座標

```ts
const setImgAxis = (axis: InterfaceAxis) => {
  const style = translateStyle(
    {
      scale: LayoutContainer.imgAxis.scale,
      imgStyle: { ...LayoutContainer.imgLayout },
      layoutStyle: { ...LayoutContainer.wrapLayout },
      rotate: LayoutContainer.imgAxis.rotate,
    },
    axis,
  )
  LayoutContainer.imgExhibitionStyle = style.imgExhibitionStyle
  LayoutContainer.imgAxis = style.imgAxis
  queueRealTimeEmit()
}
```

呼叫 `translateStyle` 重新計算 CSS transform，更新 `imgExhibitionStyle`，Vue 的響應式系統會自動重繪。

---

## 縮放

### setScale：設定縮放比例

```ts
const setScale = (scale: number, keep: boolean = false) => {
  const axis = {
    x: LayoutContainer.imgAxis.x,
    y: LayoutContainer.imgAxis.y,
  }

  if (!keep) {
    // 縮放時維持圖片中心不動（從中心縮放）
    axis.x -= (LayoutContainer.imgLayout.width * (scale - LayoutContainer.imgAxis.scale)) / 2
    axis.y -= (LayoutContainer.imgLayout.height * (scale - LayoutContainer.imgAxis.scale)) / 2
  }
```

**為什麼縮放時要調整 x、y？**

`setScale` 改變的是圖片的縮放比例，但圖片的「定位點」是左上角。

如果只改 scale 不動 x、y，圖片會從左上角縮放（左上角固定，右下角縮回），看起來很奇怪。

我們想要的效果是「從中心縮放」：縮小時，圖片各邊等距收縮；放大時，各邊等距擴張。

數學推導：
```
放大前：圖片中心 = x + width/2
放大後：圖片中心 = x' + (scale * width / 2)
若要中心不動：x + width/2 = x' + scale * width / 2
→ x' = x + width/2 - scale * width / 2
→ x' = x - width * (scale - 1) / 2
→ 調整量 = -width * (scale - oldScale) / 2
```

```ts
  const style = translateStyle(
    { scale, imgStyle: { ...LayoutContainer.imgLayout },
      layoutStyle: { ...LayoutContainer.wrapLayout }, rotate: LayoutContainer.imgAxis.rotate },
    axis,
  )
  LayoutContainer.imgExhibitionStyle = style.imgExhibitionStyle
  LayoutContainer.imgAxis = style.imgAxis
  queueRealTimeEmit()

  // 縮放完成後延遲檢查邊界
  if (setWaitFunc.value !== null) clearTimeout(setWaitFunc.value)
  const boundaryDuration = getBoundaryDuration()
  setWaitFunc.value = setTimeout(() => {
    reboundImg()
  }, boundaryDuration)
}
```

**為什麼縮放後要延遲 `reboundImg`，而移動後可以立刻？**

移動時，使用者放開滑鼠才回彈（`cropImg.on('up', reboundImg)`）。

縮放（滾輪）沒有「放開」的概念，所以用 `setTimeout` 在縮放停止後短暫延遲再回彈。每次滾輪事件都會重設這個計時器（`clearTimeout` + 重新設定），所以只要持續滾動，就不會觸發回彈。

---

### 滑鼠滾輪縮放

```ts
const mouseInCropper = () => {
  if (isIE) {
    window.addEventListener(supportWheel, mouseScroll)
  } else {
    window.addEventListener(supportWheel, mouseScroll, { passive: false })
  }
}

const mouseOutCropper = () => {
  window.removeEventListener(supportWheel, mouseScroll)
}

const mouseScroll = (e: Event) => {
  e.preventDefault()
  const scale = changeImgSize(e, LayoutContainer.imgAxis.scale, LayoutContainer.imgLayout)
  setScale(scale)
}
```

**進入元件範圍才監聽滾輪，離開後移除**。

為什麼不在元件上直接監聽（`@wheel`）？

因為 `@wheel` 綁在元素上只有在元素可見範圍內有效，而且預設是 passive（無法 `preventDefault()`）。

綁在 `window` 上、在 mouseover/mouseout 之間，可以確保即使滑鼠在容器邊緣，滾輪依然有效。

---

### 觸控雙指縮放

```ts
const moveScale = (message: InterfaceMessageEvent) => {
  isImgTouchScale.value = true  // 標記「正在雙指縮放」
  if (message.scale) {
    const scale = changeImgSizeByTouch(message.scale, LayoutContainer.imgAxis.scale)
    setScale(scale)
  }
}
```

`isImgTouchScale.value = true` 在雙指縮放時禁止裁切框移動（防止縮放和移動同時觸發）。

---

## 回彈：reboundImg

```ts
const reboundImg = (): void => {
  isImgTouchScale.value = false  // 解除雙指縮放鎖定

  if (!centerBox.value && !centerWrapper.value) return  // 沒開啟限制，不需要回彈

  const boundaryDuration = getBoundaryDuration()
  let crossing

  if (centerBox.value) {
    crossing = detectionBoundary(
      { ...LayoutContainer.cropAxis },
      { ...effectiveCropLayoutStyle.value },
      { ...LayoutContainer.imgAxis },
      { ...LayoutContainer.imgLayout },
    )
  } else {
    // centerWrapper：限制在容器（不是裁切框）內
    crossing = detectionBoundary(
      { x: 0, y: 0 },
      { ...LayoutContainer.wrapLayout },
      { ...LayoutContainer.imgAxis },
      { ...LayoutContainer.imgLayout },
    )
  }

  // 如果圖片太小（縮得比裁切框還小），需要先放大
  if (LayoutContainer.imgAxis.scale < crossing.scale) {
    setAnimation(LayoutContainer.imgAxis.scale, crossing.scale, boundaryDuration, value => {
      setScale(value, true)  // keep=true：縮放時不調整座標（回彈不需要從中心縮放）
    })
  }

  // 水平方向回彈
  if (crossing.landscape === 'left') {
    setAnimation(LayoutContainer.imgAxis.x, crossing.boundary.left, boundaryDuration, value => {
      setImgAxis({ x: value, y: LayoutContainer.imgAxis.y })
    })
  }
  if (crossing.landscape === 'right') {
    setAnimation(LayoutContainer.imgAxis.x, crossing.boundary.right, boundaryDuration, value => {
      setImgAxis({ x: value, y: LayoutContainer.imgAxis.y })
    })
  }

  // 垂直方向回彈
  if (crossing.portrait === 'top') {
    setAnimation(LayoutContainer.imgAxis.y, crossing.boundary.top, boundaryDuration, value => {
      setImgAxis({ x: LayoutContainer.imgAxis.x, y: value })
    })
  }
  if (crossing.portrait === 'bottom') {
    setAnimation(LayoutContainer.imgAxis.y, crossing.boundary.bottom, boundaryDuration, value => {
      setImgAxis({ x: LayoutContainer.imgAxis.x, y: value })
    })
  }

  queueRealTimeEmit()
}
```

**回彈流程**：

1. `detectionBoundary` 回傳「超出了哪個方向、應該在哪個座標」
2. 用 `setAnimation` 做緩動動畫，從當前值滑到目標值
3. 每一幀的回調都呼叫 `setImgAxis`，更新圖片位置

---

## 旋轉

```ts
const setRotate = (rotate: number, shouldRebound: boolean = true) => {
  const { x, y, scale } = LayoutContainer.imgAxis
  const style = translateStyle(
    { scale, imgStyle: { ...LayoutContainer.imgLayout },
      layoutStyle: { ...LayoutContainer.wrapLayout }, rotate },
    { x, y },
  )
  LayoutContainer.imgExhibitionStyle = style.imgExhibitionStyle
  LayoutContainer.imgAxis = style.imgAxis
  queueRealTimeEmit()

  // 旋轉改變圖片的有效邊界，需要重新做邊界校驗
  if (shouldRebound && imgs.value) {
    reboundImg()
  }
}

// 快捷函數（給父元件呼叫）
const rotateLeft = () => { setRotate(LayoutContainer.imgAxis.rotate - 90) }
const rotateRight = () => { setRotate(LayoutContainer.imgAxis.rotate + 90) }
const rotateClear = () => { setRotate(0) }
```

**為什麼旋轉後需要 `reboundImg`？**

旋轉後圖片的有效覆蓋範圍改變了（例如一張橫向圖片轉 45° 後，對角線更長，可能不再能覆蓋裁切框）。

需要立刻重新計算邊界，如果圖片太小了就自動放大。

---

## 綁定/解綁函數

### bindMoveImg / unbindMoveImg

```ts
const bindMoveImg = (): void => {
  unbindMoveImg()  // 先解綁，避免重複綁定
  const domImg = cropperImg.value
  cropImg = new TouchEvent(domImg)
  cropImg.on('down-to-move', moveImg)
  cropImg.on('down-to-scale', moveScale)
  cropImg.on('up', reboundImg)
}

const unbindMoveImg = (): void => {
  if (cropImg) {
    cropImg.off('down-to-move', moveImg)
    cropImg.off('up', reboundImg)
    cropImg.off('down-to-scale', moveScale)
  }
}
```

**先 `unbind` 再 `bind` 的原因**：

圖片 URL 改變時，`watch(imgs)` 重新呼叫 `bindMoveImg`。如果不先解綁，同一個 DOM 元素會有兩組事件監聽器，每次滑鼠移動都觸發兩次 `moveImg`，圖片移動速度會變成兩倍。

### bindMoveCrop / unbindMoveCrop

```ts
const bindMoveCrop = (): void => {
  unbindMoveCrop()
  const domBox = cropperBox.value
  if (!domBox) return
  cropBox = new TouchEvent(domBox)
  cropBox.on('down-to-move', moveCrop)
  cropBox.on('down-to-scale', moveScale)
  cropBox.on('up', reboundImg)
  cropImg = null  // 裁切框出現時，圖片不再單獨可拖（避免衝突）
}
```

---

## 小結

這篇涵蓋了裁剪元件的所有互動邏輯：

| 操作 | 函數鏈 |
|------|--------|
| 圖片拖曳 | `moveImg` → `setImgAxis` → `translateStyle` |
| 滾輪縮放 | `mouseScroll` → `changeImgSize` → `setScale` → `translateStyle` |
| 雙指縮放 | `moveScale` → `changeImgSizeByTouch` → `setScale` → `translateStyle` |
| 放開回彈 | `reboundImg` → `detectionBoundary` → `setAnimation` → `setImgAxis` |
| 旋轉 | `setRotate` → `translateStyle` → `reboundImg` |

下一篇：[16 — lib/vue-cropper.vue Part 4：裁切框操作與導出](./16-main-comp-4-crop.md)
