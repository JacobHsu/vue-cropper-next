# 12 — lib/style/index.scss（樣式設計）

## 為什麼用 SCSS 而不是純 CSS？

SCSS 是 CSS 的超集（所有合法的 CSS 都是合法的 SCSS），它新增了幾個讓 CSS 更容易維護的功能：

- **巢狀選擇器**（Nesting）：把相關的樣式放在一起，不用重複寫父選擇器
- **變數**：儲存顏色、尺寸等值，集中管理
- **函數**：如 `rgba($color, $alpha)`

打包時，SCSS 會被編譯成普通的 CSS。

---

## 建立 lib/style/index.scss

### CSS 自訂屬性：主題色

```scss
:root {
  --cropperColor: #1890ff;
}
```

**CSS 自訂屬性（CSS Variables）**以 `--` 開頭。

這個設計讓使用者可以直接修改主題色，不需要覆蓋深層的 class：

```css
/* 使用者在自己的樣式表裡 */
:root {
  --cropperColor: #ff6b6b;  /* 改成紅色 */
}
```

這個顏色目前用在 loading 旋轉圖示的顏色（`color: var(--cropperColor)`）。

---

### 外層容器：.vue-cropper

```scss
.vue-cropper {
  position: relative;   /* 讓絕對定位的子元素以此為基準 */
  box-sizing: border-box;
  direction: ltr;       /* 強制從左到右排列（支援 RTL 語言的環境）*/
  touch-action: none;   /* 禁止瀏覽器的預設觸控行為（滾動、縮放）*/
  text-align: left;
  background: url('data:image/png;base64,...'); /* 棋盤格背景 */
  user-select: none;    /* 禁止文字選取（避免拖曳時反白）*/
```

**`touch-action: none`**：非常重要。如果不設定，在手機上拖曳圖片時，整個頁面也會跟著滾動，體驗很差。這個屬性告訴瀏覽器「這個元素的觸控事件由 JavaScript 處理，不要預設行為」。

**棋盤格背景**：這是一個 Base64 編碼的 PNG 圖片，顯示棋盤格（灰白交替的方格）。

棋盤格是業界慣例，用來表示**透明區域**。因為透明本身沒有視覺效果，用棋盤格讓使用者知道「這裡是透明的，不是白色的」。

---

### 子元素共用樣式

```scss
.cropper-box,
.cropper-box-canvas,
.cropper-drag-box,
.cropper-crop-box,
.cropper-face {
  position: absolute;  /* 絕對定位，疊在容器裡 */
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;             /* 四邊都設為 0 = 填滿父容器 */
  user-select: none;
}
```

**為什麼所有子元素都要 `position: absolute` 並填滿父容器？**

這個設計讓所有子元素**疊在同一個位置**，像圖層一樣：

```
.vue-cropper（容器）
  └── .cropper-box（圖片層）         ← absolute, 填滿
       └── .cropper-box-canvas（圖片）
       └── .cropper-drag-box（拖曳感應層）← 透明，捕捉事件
  └── .cropper-crop-box（裁切框層）  ← absolute, 用 transform 定位
  └── .cropper-loading（loading 層） ← absolute, 覆蓋在最上面
```

---

### 圖片容器：overflow: hidden

```scss
.cropper-box {
  overflow: hidden;  /* 圖片超出容器的部分被裁掉 */
}
```

圖片可能比容器大（尤其是 cover 模式），`overflow: hidden` 確保超出的部分不可見。

---

### 拖曳感應層

```scss
.cropper-drag-box {
  transition: background 0.2s ease-in-out;
}
.cropper-modal {
  background: rgba(0, 0, 0, 0.5);  /* 半透明黑色遮罩 */
}
```

`.cropper-drag-box` 是一個透明的覆蓋層，負責捕捉滑鼠/觸控事件（讓圖片可以被拖曳）。

當裁切框開啟（`.cropper-modal` class 被加上）時，背景變成半透明黑色，讓裁切框外的圖片看起來「暗掉」，突顯裁切範圍。

這個 class 切換在 `vue-cropper.vue` 裡：

```ts
const computedClassDrag = (): string => {
  const className = ['cropper-drag-box']
  if (cropping.value) {
    className.push('cropper-modal')  // 有裁切框時加上遮罩
  }
  return className.join(' ')
}
```

---

### 裁切框顯示層

```scss
.cropper-view-box {
  display: block;
  overflow: hidden;   /* 超出裁切框的圖片部分不顯示 */
  width: 100%;
  height: 100%;
  outline: 1px solid #fff;
  outline-color: #fff;  /* 白色外框，可被 cropColor prop 覆蓋 */
  user-select: none;
}
.cropper-view-box img {
  max-width: none;    /* 移除瀏覽器預設的 max-width: 100% */
  max-height: none;   /* 讓圖片能放大超過容器 */
}
```

**`.cropper-view-box`** 是裁切框裡那個「看起來圖片在這裡是清楚的」的層。它是一個 `overflow: hidden` 的框，裡面有一張和主圖完全相同的圖片，但用 `transform` 讓它的可見範圍剛好和裁切框對齊。

這個「雙圖層」技術讓框外顯示暗（遮罩），框內顯示亮（原始圖片），視覺上形成裁切框效果。

**`max-width: none`** 很重要：瀏覽器預設 `<img>` 有 `max-width: 100%`，這會讓大圖片縮小到容器內。我們需要手動覆蓋，讓圖片可以比容器大。

---

### 裁切框的透明面板

```scss
.cropper-face {
  top: 0;
  left: 0;
  background-color: #fff;
  opacity: 0.1;  /* 幾乎全透明的白色覆蓋層 */
}
```

`.cropper-face` 是裁切框上的一層幾乎全透明的白色覆蓋，讓裁切框看起來比背景稍微「亮一點」，增加辨識度。

---

### Loading 樣式

```scss
.cropper-loading {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  background-color: rgba($color: #fff, $alpha: 0.8);  /* 半透明白色遮罩 */
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--cropperColor);  /* 使用主題色 */

  .cropper-loading-spin {
    /* SVG 旋轉動畫樣式 */
    i svg {
      animation: cropperLoadingCircle 1.2s infinite ease-out;
    }
  }
}
```

**SCSS 巢狀語法**：`.cropper-loading` 裡面的 `.cropper-loading-spin` 不需要重複寫 `.cropper-loading .cropper-loading-spin`，直接巢狀即可。

`rgba($color: #fff, $alpha: 0.8)` 是 SCSS 的 `rgba()` 函數語法（用具名參數），等同 CSS 的 `rgba(255, 255, 255, 0.8)`。

---

### 淡入動畫

```scss
.cropper-fade-in {
  animation-name: cropperFadeIn;
  animation-duration: 0.5s;
  animation-timing-function: ease-in-out;
}

@keyframes cropperFadeIn {
  from { opacity: 0; }
  to   { opacity: 1; }
}
```

圖片和裁切框出現時都有淡入效果，讓 UX 更柔和。

---

### Loading 旋轉動畫

```scss
@keyframes cropperLoadingCircle {
  100% {
    transform: rotate(360deg);
  }
}
```

從 `0%`（預設，`rotate(0)`）到 `100%`（`rotate(360°)`），加上 `infinite`，就是無限循環的旋轉。

---

### 多個裁剪元件並排

```scss
& + .vue-cropper {
  margin-left: 20px;
}
```

`&` 是 SCSS 的父選擇器符號，`& + .vue-cropper` 等同 `.vue-cropper + .vue-cropper`。

意思：當兩個 `.vue-cropper` 相鄰時，第二個自動加上 20px 左間距。

---

## 完整的 lib/style/index.scss

直接參考 [lib/style/index.scss](../lib/style/index.scss)。

---

## 小結

這個樣式檔設計了兩個核心機制：

1. **多層疊加**（all absolute + 填滿）：讓圖片層、拖曳感應層、裁切框層、loading 層可以獨立控制，互不干擾
2. **雙圖層技術**：主圖片 + 遮罩 + 裁切框內的同一張圖片，視覺上形成「框內亮、框外暗」的裁切框效果

下一篇：[13 — lib/vue-cropper.vue Part 1：Props 定義與狀態結構](./13-main-comp-1-props.md)
