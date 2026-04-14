# 04 — lib/interface.ts + lib/config.ts

## 先寫型別，再寫邏輯

`interface.ts` 是整個 lib/ 的**地基**。它只有型別定義，沒有任何執行邏輯。

之所以要先寫它，是因為後續所有檔案（`common.ts`、`touch/index.ts`、`vue-cropper.vue`）都需要引用這些型別。先定義型別，TypeScript 才能在你寫其他函數時，立刻提醒你「參數形狀不對」。

---

## 建立 lib/interface.ts

```ts
import type { CSSProperties } from 'vue'
```

> **TypeScript 穿插說明 — `import type`**  
> `import type` 表示「只引用型別，不引用任何值」。這樣打包工具在處理 JS 時可以完全移除這行，不影響執行期大小。  
> 這裡引用 Vue 的 `CSSProperties`，它是 Vue 定義的型別，描述「合法的 CSS 屬性物件」（如 `{ width: '100px', color: 'red' }`）。

---

### `declare global`：擴充全域型別

```ts
declare global {
  interface Window {
    requestAnimationFrame: any,
    ActiveXObject?: unknown,
  }
}
```

> **TypeScript 穿插說明 — `declare global`**  
> TypeScript 對瀏覽器的全域物件（`window`、`document`）都有預設的型別定義。  
> 但有時我們需要「新增」或「修改」這些定義：  
> - `requestAnimationFrame`：確保在所有瀏覽器環境下都被承認為 `window` 的屬性
> - `ActiveXObject`：IE 的特有屬性，加上 `?` 表示可選（非 IE 瀏覽器沒有這個）
>
> `unknown` 比 `any` 更安全——你不能直接對 `unknown` 做任何操作，必須先做型別判斷。

---

### `InterfaceLength`：寬鬆的長度型別

```ts
export type InterfaceLength = number | string
```

為什麼長度要同時支援 `number` 和 `string`？

因為 CSS 的長度可以是：
- `300`（數字，會被自動轉成 `300px`）
- `'300px'`（字串，直接使用）
- `'50%'`（百分比，不能用純數字表示）

這個型別讓使用者在傳入 Props 時可以靈活選擇：

```html
<VueCropper :wrapper="{ width: 300, height: '50%' }" />
```

> **TypeScript 穿插說明 — `type` vs `interface`**  
> `type` 用來定義**聯合型別**（`A | B`）、交叉型別（`A & B`）、原始型別別名。  
> `interface` 用來定義**物件形狀**（有哪些屬性）。  
> 基本規則：物件形狀用 `interface`，其他用 `type`。

---

### `InterfaceLayout`：容器尺寸型別

```ts
export interface InterfaceLayout extends CSSProperties {
  width: InterfaceLength
  height: InterfaceLength
  background?: string
  backgroundImage?: string
}
```

> **TypeScript 穿插說明 — `extends`**  
> `extends CSSProperties` 表示「繼承 CSSProperties 的所有屬性，再加上我自己定義的」。  
> 效果：`InterfaceLayout` 除了 `width`、`height`，還可以放任何合法的 CSS 屬性（如 `border`、`overflow` 等）。

這個型別是給 `wrapper` prop 用的，讓使用者可以自訂外層容器的任何樣式，但 `width` 和 `height` 是必填的。

---

### `InterfaceLayoutInput`：只有尺寸，沒有樣式

```ts
export interface InterfaceLayoutInput {
  width: InterfaceLength
  height: InterfaceLength
}
```

這個給 `cropLayout`（裁切框大小）用。裁切框只需要尺寸，不需要其他 CSS 屬性，所以用比 `InterfaceLayout` 更精簡的版本。

---

### `InterfaceImgLoad`：圖片載入結果

```ts
export interface InterfaceImgLoad {
  type: string
  message: string
}
```

這是 `img-load` 事件的 payload。例如：

```js
{ type: 'success', message: '圖片加載成功' }
{ type: 'error', message: '圖片加載失敗...' }
```

---

### `InterfaceRealTimePreview`：即時預覽資料

```ts
export interface InterfaceRealTimePreview {
  w: number           // 裁切框寬度
  h: number           // 裁切框高度
  url: string         // 圖片的目前 URL；在這個專案裡通常是 Blob URL
  img: {
    width: string     // 圖片 CSS 寬度（含 px）
    height: string    // 圖片 CSS 高度（含 px）
    transform: string // 圖片 CSS transform（scale + translate + rotate）
  }
  html: string        // 可以直接塞進 innerHTML 的 HTML 字串
}
```

這是 `real-time` 事件的 payload，讓使用者在畫面旁邊顯示「裁切後的預覽」。

---

### `InterfaceLayoutStyle`：純數字尺寸

```ts
export interface InterfaceLayoutStyle {
  width: number
  height: number
}
```

注意這裡的 `width` / `height` 是純 `number`（無單位），不像 `InterfaceLength` 可以是字串。

這個型別用在**內部計算**，所有尺寸在進入計算邏輯前都必須先轉成純數字。

---

### `InterfaceModeHandle`：圖片佈局模式

```ts
export interface InterfaceModeHandle {
  contain: () => {}
  cover: () => {}
  default: () => {}
}
```

用途是讓 TypeScript 知道 `mode` prop 只能是 `'contain'`、`'cover'`、`'default'` 三個值之一：

```ts
mode?: keyof InterfaceModeHandle
// 等同於：mode?: 'contain' | 'cover' | 'default'
```

> **TypeScript 穿插說明 — `keyof`**  
> `keyof SomeInterface` 取得這個 interface 所有屬性名稱組成的聯合型別。  
> `keyof InterfaceModeHandle` = `'contain' | 'cover' | 'default'`  
> 這樣如果你傳入 `mode="zoom"`，TypeScript 會立刻報錯。

---

### `InterfaceAxis`：二維座標

```ts
export interface InterfaceAxis {
  x: number
  y: number
}
```

整個專案最常用的型別。所有「位置」都用這個表示：圖片左上角座標、裁切框座標、滑鼠移動量。

---

### `InterfaceImgAxis`：圖片完整狀態

```ts
export interface InterfaceImgAxis extends InterfaceAxis {
  scale: number   // 縮放比例（1 = 原始大小）
  rotate: number  // 旋轉角度（度數，正數為順時針）
}
```

繼承了 `InterfaceAxis`（有 `x`、`y`），再加上 `scale` 和 `rotate`。

這個型別記錄了圖片的「完整狀態」：在哪裡、放多大、轉幾度。

---

### `InterfaceTransformStyle`：CSS transform 字串

```ts
export interface InterfaceTransformStyle extends CSSProperties {
  width: string
  height: string
  transform: string
}
```

用來描述圖片和裁切框的 CSS 樣式物件，會直接 `:style` 綁定到 template。

---

### `InterfaceBoundary`：邊界計算結果

```ts
export interface InterfaceBoundary {
  left: number    // 圖片允許的最左 x 座標
  right: number   // 圖片允許的最右 x 座標
  top: number     // 圖片允許的最上 y 座標
  bottom: number  // 圖片允許的最下 y 座標
  scale: number   // 圖片需要的最小縮放比例
}
```

當 `centerBox: true` 時，圖片不能拖出裁切框範圍。這個型別記錄「圖片可以在的邊界」，超出就要回彈。

---

### `InterfaceMessageEvent`：事件訊息

```ts
export interface InterfaceMessageEvent {
  type: string
  event?: Event
  change?: InterfaceAxis
  scale?: number
}
```

這是觀察者模式（`watchEvents.ts`）傳遞的訊息格式：

| `type` | 說明 | 附帶資料 |
|--------|------|---------|
| `'down'` | 按下滑鼠/觸控 | `event` |
| `'down-to-move'` | 拖曳移動 | `change`（移動量）|
| `'down-to-scale'` | 雙指縮放 | `scale`（縮放比例）|
| `'up'` | 放開滑鼠/觸控 | `event` |

---

## 完整的 lib/interface.ts

```ts
import type { CSSProperties } from 'vue'

declare global {
  interface Window {
    requestAnimationFrame: any,
    ActiveXObject?: unknown,
  }
}

export type InterfaceLength = number | string

export interface InterfaceLayout extends CSSProperties {
  width: InterfaceLength
  height: InterfaceLength
  background?: string
  backgroundImage?: string
}

export interface InterfaceLayoutInput {
  width: InterfaceLength
  height: InterfaceLength
}

export interface InterfaceImgLoad {
  type: string
  message: string
}

export interface InterfaceRealTimePreview {
  w: number
  h: number
  url: string
  img: {
    width: string
    height: string
    transform: string
  }
  html: string
}

export interface InterfaceLayoutStyle {
  width: number
  height: number
}

export interface InterfaceModeHandle {
  contain: () => {}
  cover: () => {}
  default: () => {}
}

export interface InterfaceRenderImgLayout {
  scale: number
  rotate: number
  imgStyle: InterfaceLayoutStyle
  layoutStyle: InterfaceLayoutStyle
}

export interface InterfaceMessageEvent {
  type: string
  event?: Event
  change?: InterfaceAxis,
  scale?: number,
}

export interface InterfaceAxis {
  x: number
  y: number
}

export interface InterfaceImgAxis extends InterfaceAxis {
  scale: number
  rotate: number
}

export interface InterfaceTransformStyle extends CSSProperties {
  width: string
  height: string
  transform: string
}

export interface InterfaceBoundary {
  left: number
  right: number
  top: number
  bottom: number
  scale: number
}
```

---

## 建立 lib/config.ts

```ts
/**
 * 配置表
 * 渲染控制
 * 全局参数
 */

// 阻力系數：圖片超出限制區域拖曳時的阻力（0.2 = 只移動 20%）
export const RESISTANCE: number = 0.2

// 回彈的時間（毫秒）
export const BOUNDARY_DURATION = 100
```

### RESISTANCE（阻力係數）

數值 `0.2` 的意義：當圖片被拖出裁切框邊界，滑鼠移動 100px，圖片只移動 20px。

這創造了「橡皮筋感」——你能感覺到阻力，知道已經超出範圍，放開後會自動彈回。

如果改成 `0`，圖片就完全拖不出去（硬擋）；改成 `1`，就沒有阻力（感覺怪）。

### BOUNDARY_DURATION（回彈時間）

`100` 毫秒是回彈動畫的持續時間。這個值偏短（0.1 秒），讓回彈看起來「果斷俐落」而不是「慢慢飄回來」。

這兩個值抽出來放在 `config.ts` 的好處：以後要調整「手感」，只改這一個檔案就好，不用到處找。

---

## 小結

這兩個檔案完成後，你建立了：

- **型別系統**：整個專案的資料形狀都有名字了
- **常數**：可調整的參數集中管理

之後每當你在某個函數裡看到 `InterfaceAxis`、`InterfaceLayoutStyle` 等型別，回來這篇查就知道它是什麼形狀。

下一篇：[05 — lib/watchEvents.ts（觀察者模式）](./05-watch-events.md)
