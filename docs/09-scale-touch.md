# 09 — lib/changeImgSize.ts + lib/touch/index.ts（縮放與觸控）

## 這篇解決什麼問題？

使用者操作圖片有兩種縮放方式：
1. **滑鼠滾輪**（桌面電腦）
2. **雙指捏合**（手機/平板，Pinch to Zoom）

同時，所有的拖曳操作（圖片移動、裁切框移動）也要同時支援滑鼠和觸控。

這篇介紹兩個檔案：
- `changeImgSize.ts`：處理「縮放量的計算」
- `touch/index.ts`：統一封裝滑鼠和觸控事件，讓 `vue-cropper.vue` 不需要分別處理兩套 API

---

## lib/changeImgSize.ts

### 瀏覽器相容問題：wheel 事件

不同瀏覽器的滾輪事件名稱不同：

```ts
export const supportWheel =
  typeof document !== 'undefined' && 'onwheel' in document
    ? 'wheel'
    : 'mousewheel'
```

現代瀏覽器都支援 `wheel`，舊版 IE/Safari 用 `mousewheel`。這個判斷確保兩者都能正常運作。

---

### isIE：偵測 IE 瀏覽器

```ts
export const isIE =
  typeof window !== 'undefined' && (!!window.ActiveXObject || 'ActiveXObject' in window)
```

IE 的 `deltaY` 符號和其他瀏覽器**相反**（正負號不同），需要特別處理。

---

### changeImgSize：計算滾輪縮放後的新 scale

```ts
let coe = 0.2           // 每像素對應的縮放係數
let coeStatus = ''      // 用來追蹤縮放方向（'+'放大 或 '-'縮小）
let scaling = false     // 是否正在縮放（用來實現慣性）
```

```ts
export const changeImgSize = (e: any, scale: number, imgStyle: InterfaceLayoutStyle): number => {
  let change = e.deltaY || e.wheelDelta   // 取滾輪變化量
  let nowScale: number = scale

  // Firefox 的 deltaY 單位是行數，需要乘以 30 才跟其他瀏覽器一致
  if (isFirefox) {
    change = change * 30
  }
  // IE 的符號相反，取反
  if (isIE) {
    change = -change
  }

  // 縮放係數：1px 圖片寬對應 0.2 的變化量
  // 圖片越小，滾輪的縮放感越靈敏；圖片越大，縮放感越平緩
  const nowCoe = coe / imgStyle.width
  const num = nowCoe * change

  if (num < 0) {
    nowScale += Math.abs(num)   // change < 0：往上滾 → 放大
  }
  if (num > 0 && scale > Math.abs(num)) {
    nowScale -= Math.abs(num)   // change > 0：往下滾 → 縮小（但不能縮到 0）
  }
```

**`coe / imgStyle.width` 的設計**：

縮放係數和圖片寬度成反比。這樣設計的效果：
- 小圖片（寬度 100px）：`nowCoe = 0.2 / 100 = 0.002`，每像素滾輪 scale 變化 0.002
- 大圖片（寬度 1000px）：`nowCoe = 0.2 / 1000 = 0.0002`，每像素滾輪 scale 變化 0.0002

結果：不管圖片多大，縮放的「感覺」（視覺上的變化速度）大致相同。

---

### 慣性縮放邏輯

```ts
  // 連續同方向滾動：係數漸增（讓連續滾動更快）
  if (num < 0 && coeStatus !== '+') {
    coe = 0.2
    coeStatus = '+'
  } else if (num < 0) {
    coe += 0.01
  }

  if (num > 0 && coeStatus !== '-') {
    coe = 0.2
    coeStatus = '-'
  } else if (num > 0) {
    coe += 0.01
  }
```

連續往同一方向滾動時，係數從 0.2 慢慢增加，讓「快速滾動」感覺更有力。

---

### changeImgSizeByTouch：計算雙指縮放後的新 scale

```ts
export const changeImgSizeByTouch = (scale: number, nowScale: number): number => {
  return nowScale * scale
}
```

觸控縮放更簡單：`touch/index.ts` 會計算兩指之間的距離比例（本次距離 / 上次距離），這個比例直接和當前 scale 相乘就是新的 scale。

---

## lib/touch/index.ts

### 設計目標

這個類別的目標是：讓 `vue-cropper.vue` 可以用**統一的介面**監聽「拖曳」和「縮放」，不需要分別處理滑鼠和觸控：

```ts
// vue-cropper.vue 的用法：
const cropImg = new CropperTouchEvent(domImg)
cropImg.on('down-to-move', moveImg)    // 不管滑鼠還是觸控，都觸發
cropImg.on('down-to-scale', moveScale) // 雙指縮放觸發
cropImg.on('up', reboundImg)           // 放開觸發
```

---

### 類別結構

```ts
class CropperTouchEvent {
  element: HTMLElement          // 綁定事件的 DOM 元素
  pre: InterfaceAxis            // 上一幀的座標（計算移動量用）
  watcher: WatchEvent           // 事件總線（觀察者模式）
  touches: TouchPointPair | null // 觸控時的兩個觸控點

  constructor(element: HTMLElement) {
    this.element = element
    this.touches = null
    this.pre = { x: 0, y: 0 }
    this.watcher = new WatchEvent()
  }
```

> **TypeScript 穿插說明 — 型別別名**  
> `type TouchPointPair = [globalThis.Touch, globalThis.Touch]`  
> 這是一個**元組型別（Tuple）**：長度固定為 2 的陣列，第一個是 Touch，第二個也是 Touch。  
> 用元組而不是 `Touch[]`（陣列），是因為我們確定剛好有兩個觸控點（雙指）。

---

### 滑鼠事件流程

```
滑鼠按下 (mousedown) → start()
    ↓
滑鼠移動 (mousemove on window) → move()
    ↓ 每移動一次
fire({ type: 'down-to-move', change: { x, y } })
    ↓
滑鼠放開 (mouseup on window) → stop()
    ↓
fire({ type: 'up' })
```

**為什麼 mousemove 和 mouseup 綁在 window 而不是 element？**

如果綁在 element 上，滑鼠移出元素邊界後就會停止觸發，導致拖曳「卡住」。

綁在 window 上，即使滑鼠移到元素外面，只要不放開按鈕，拖曳就持續進行。

```ts
start(event: MouseEvent) {
  event.preventDefault()
  if (event.which !== 1) return  // 只處理左鍵（which=1）

  this.pre = this.getAxis(event)  // 記錄起始座標
  this.fire({ type: 'down', event })

  window.addEventListener('mousemove', this.move)  // 在 window 監聽
  window.addEventListener('mouseup', this.stop)
}

move(event: MouseEvent) {
  event.preventDefault()
  const nowAxis = this.getAxis(event)
  this.fire({
    type: 'down-to-move',
    event,
    change: {
      x: nowAxis.x - this.pre.x,   // 移動量（差值，不是絕對座標）
      y: nowAxis.y - this.pre.y,
    },
  })
  this.pre = nowAxis  // 更新上一幀座標
}

stop(event: MouseEvent) {
  this.fire({ type: 'up', event })
  window.removeEventListener('mousemove', this.move)  // 清理！
  window.removeEventListener('mouseup', this.stop)
}
```

---

### 觸控事件流程

觸控需要額外處理「單指（拖曳）」和「雙指（縮放）」的分支：

```
觸控開始 (touchstart) → startTouch()
    ↓
  單指？ → 監聽 touchmove → moveTouch() → fire('down-to-move')
  雙指？ → 監聽 touchmove → scaleTouch() → fire('down-to-scale')
    ↓
觸控結束 (touchend) → stopTouch()
    ↓
fire('up')
```

```ts
startTouch(event: globalThis.TouchEvent) {
  const touch = event.touches[0]
  if (!touch) return

  this.pre = { x: touch.clientX, y: touch.clientY }
  this.fire({ type: 'down', event })

  const touchPair = this.getTouchPair(event.touches)
  if (touchPair) {
    // 雙指：切換到縮放模式
    this.touches = touchPair
    window.addEventListener('touchmove', this.scaleTouch, { passive: false })
    window.removeEventListener('touchmove', this.moveTouch)
  } else {
    // 單指：拖曳模式
    window.addEventListener('touchmove', this.moveTouch, { passive: false })
    window.removeEventListener('touchmove', this.scaleTouch)
  }
  window.addEventListener('touchend', this.stopTouch)
}
```

**`{ passive: false }`**：預設 touch 事件是 passive（讓瀏覽器提前滾動頁面，提升流暢度）。加上 `passive: false` 才能在事件處理函數裡呼叫 `event.preventDefault()`，防止觸控拖曳時頁面跟著滾動。

---

### 雙指縮放計算

```ts
private getLen(touches: InterfaceAxis): number {
  return Math.sqrt(touches.x * touches.x + touches.y * touches.y)
}

private getScale(cur: TouchPointPair, pre: TouchPointPair): number {
  const curCenter = {
    x: cur[1].clientX - cur[0].clientX,
    y: cur[1].clientY - cur[0].clientY,
  }
  const preCenter = {
    x: pre[1].clientX - pre[0].clientX,
    y: pre[1].clientY - pre[0].clientY,
  }
  return this.getLen(curCenter) / this.getLen(preCenter)
}
```

計算兩個觸控點的向量（`cur[1] - cur[0]`），取其**長度（兩點距離）**的比值：

```
scale = 本幀兩指距離 / 上幀兩指距離
```

- `scale = 1.1`：兩指分開了 10% → 放大 10%
- `scale = 0.9`：兩指靠近了 10% → 縮小 10%

---

### on 方法：訂閱 + 綁定 DOM 事件

```ts
on(type: string, handler: (message: InterfaceMessageEvent) => void) {
  this.watcher.addHandler(type, handler)  // 訂閱事件總線

  // 只在第一次訂閱時才綁定 DOM 事件
  if (type !== 'down-to-move' && type !== 'down-to-scale' && type !== 'up') {
    return
  }

  if (SUPPORT_MOUSE) {
    this.start = this.start.bind(this)
    // ...
    this.element.addEventListener('mousedown', this.start)
  }

  if (SUPPORT_TOUCH) {
    this.startTouch = this.startTouch.bind(this)
    // ...
    this.element.addEventListener('touchstart', this.startTouch, { passive: false })
  }
}
```

**`.bind(this)` 的必要性**：

類別方法在被 `addEventListener` 呼叫時，`this` 會遺失（指向 `undefined` 或 `window`）。

`this.start.bind(this)` 建立一個新函數，其中 `this` 永遠指向這個類別的實例。

---

## 小結

| 檔案 | 職責 |
|------|------|
| `changeImgSize.ts` | 計算滾輪/觸控縮放後的新 scale 值 |
| `touch/index.ts` | 統一封裝滑鼠 + 觸控事件 → 發送統一格式的訊息 |

這兩個模組配合觀察者模式（`watchEvents.ts`），讓 `vue-cropper.vue` 只需要監聽「語意事件」（`'down-to-move'`、`'up'`），不需要關心底層是滑鼠還是觸控。

下一篇：[10 — lib/filter/index.ts（Canvas 濾鏡原理）](./10-filter.md)
