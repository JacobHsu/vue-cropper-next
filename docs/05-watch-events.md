# 05 — lib/watchEvents.ts（觀察者模式）

## 為什麼需要這個？

先想一個問題：觸控事件（`touchstart`、`touchmove`）和滑鼠事件（`mousedown`、`mousemove`）是兩套不同的瀏覽器 API。

裁剪元件需要同時支援兩種操作，而且**圖片**和**裁切框**各自要能獨立拖曳。

如果直接在 `vue-cropper.vue` 裡寫這些邏輯，程式會很混亂。我們需要一個方法，讓「事件的觸發」和「事件的處理」分離。

這就是**觀察者模式（Observer Pattern）**。

---

## 觀察者模式的概念

觀察者模式有兩個角色：

- **發布者**（Publisher）：知道「發生了什麼事」，負責發送通知
- **訂閱者**（Subscriber）：關心某種事件，事先登記，等通知來才執行

就像訂閱 YouTube 頻道：
- 頻道（發布者）：上傳影片時通知所有訂閱者
- 你（訂閱者）：收到通知才去看，不用一直盯著

在這個專案裡：
- `TouchEvent`（`touch/index.ts`）是發布者：偵測滑鼠/觸控事件，發送 `down-to-move`、`up` 等訊息
- `vue-cropper.vue` 是訂閱者：登記「當 `down-to-move` 發生時，執行 `moveImg` 函數」

對應到專案程式可以直接看到兩邊：

```ts
// touch/index.ts
move(event: MouseEvent) {
  this.fire({
    type: 'down-to-move',
    event,
    change: {
      x: nowAxis.x - this.pre.x,
      y: nowAxis.y - this.pre.y,
    },
  })
}
```

```ts
// vue-cropper.vue
cropImg = new TouchEvent(domImg)
cropImg.on('down-to-move', moveImg)
cropImg.on('up', reboundImg)
```

`TouchEvent` 在滑鼠/觸控事件發生時呼叫 `this.fire(...)`；`vue-cropper.vue` 用 `.on(...)` 把 `moveImg`、`reboundImg` 註冊進去。

---

## 建立 lib/watchEvents.ts

```ts
import type { InterfaceMessageEvent } from './interface'
```

引用 `InterfaceMessageEvent`，這是訊息的格式（`type`、`change`、`scale` 等）。

---

### 建立 WatchEvent 類別

> **TypeScript 穿插說明 — class**  
> TypeScript 的 `class` 和 JavaScript 的 `class` 語法相同，但可以給屬性加型別。  
> 類別屬性需要在 `constructor` 之前宣告型別：
> ```ts
> class Foo {
>   name: string   // 宣告型別
>   constructor() {
>     this.name = ''  // 賦值
>   }
> }
> ```

```ts
class WatchEvent {
  handlers: Map<string, Array<(message: InterfaceMessageEvent) => void>>
  
  constructor() {
    this.handlers = new Map()
  }
```

> **TypeScript 穿插說明 — Map<K, V>（泛型）**  
> `Map` 是 JavaScript 的內建資料結構，像一個 key-value 字典。  
> TypeScript 用泛型 `<K, V>` 指定 key 和 value 的型別：  
> `Map<string, Array<...>>` = key 是字串，value 是陣列。  
>
> 泛型（Generic）的概念：`<T>` 是一個佔位符，使用時再填入具體型別。  
> 就像函數的參數，泛型讓型別也能「傳入」。

`handlers` 是一張**事件表**：

```
"down-to-move" → [moveImg, moveOtherThing]
"up"           → [reboundImg]
"down"         → []
```

每個事件名稱對應一個**處理函數陣列**（可以有多個函數監聽同一事件）。

---

### addHandler：訂閱事件

```ts
addHandler(type: string, handler: (message: InterfaceMessageEvent) => void) {
  const res = this.handlers.get(type)
  let arr: Array<(message: InterfaceMessageEvent) => void> = []
  if (res) {
    arr = [...res]  // 複製現有陣列（不直接修改原陣列）
  }
  arr.push(handler)
  this.handlers.set(type, arr)
}
```

> **TypeScript 穿插說明 — 函數型別**  
> `(message: InterfaceMessageEvent) => void` 描述一個函數的形狀：  
> - 接受一個 `InterfaceMessageEvent` 型別的參數
> - 回傳 `void`（沒有回傳值）
>
> `void` 在 TypeScript 裡表示「這個函數的回傳值不重要，呼叫者不應該使用它」。

**為什麼用 `[...res]` 複製而不直接 push？**

```ts
// 危險（直接修改）：
arr = res  // arr 和 res 是同一個陣列
arr.push(handler)  // 同時修改了 handlers 裡的陣列

// 安全（複製）：
arr = [...res]     // arr 是新陣列
arr.push(handler)  // 只修改新陣列，再 set 回去
```

這是**不可變（immutable）**的做法。雖然在這個簡單情境下兩種都能跑，但養成習慣可以避免難以追蹤的 bug。

---

### fire：發送事件通知

```ts
// lib/watchEvents.ts
fire(event: InterfaceMessageEvent) {
  const res = this.handlers.get(event.type)
  if (!res) {
    return
  }
  res.forEach((func: (event: InterfaceMessageEvent) => void) => {
    func(event)
  })
}
```

從 `handlers` 裡找出對應 `type` 的所有處理函數，依序執行每一個。

---

### removeHandler：取消訂閱

```ts
// lib/watchEvents.ts
removeHandler(type: string, handler: (message: InterfaceMessageEvent) => void) {
  const res = this.handlers.get(type)
  if (!res) {
    return
  }
  let i = 0
  for (const len = res.length; i < len; i++) {
    if (res[i] === handler) {
      break  // 找到相同的函數就停止
    }
  }
  res.splice(i, 1)  // 移除找到的位置
  this.handlers.set(type, res)
}
```

**為什麼用 `===` 比較函數？**

JavaScript 的函數是物件，每個函數有唯一的記憶體位址。`===` 比較的是「是不是同一個函數（同一個記憶體位址）」。

這就是為什麼綁定和解綁必須用**同一個函數引用**：

```ts
// 必須這樣（同一個引用）：
const moveImg = (msg) => { ... }
cropImg.on('down-to-move', moveImg)
cropImg.off('down-to-move', moveImg)  // ✓

// 這樣無法解綁（不同函數）：
cropImg.on('down-to-move', (msg) => { ... })
cropImg.off('down-to-move', (msg) => { ... })  // ✗ 找不到
```

---

## 完整的 lib/watchEvents.ts

```ts
import type { InterfaceMessageEvent } from './interface'

class WatchEvent {
  handlers: Map<string, Array<(message: InterfaceMessageEvent) => void>>
  
  constructor() {
    this.handlers = new Map()
  }

  addHandler(type: string, handler: (message: InterfaceMessageEvent) => void) {
    const res: Array<(message: InterfaceMessageEvent) => void> | undefined = this.handlers.get(type)
    let arr: Array<(message: InterfaceMessageEvent) => void> = []
    if (res) {
      arr = [...res]
    }
    arr.push(handler)
    this.handlers.set(type, arr)
  }

  fire(event: InterfaceMessageEvent) {
    const res: Array<(message: InterfaceMessageEvent) => void> | undefined = this.handlers.get(
      event.type,
    )
    if (!res) {
      return
    }
    res.forEach((func: (event: InterfaceMessageEvent) => void) => {
      func(event)
    })
  }

  removeHandler(type: string, handler: (message: InterfaceMessageEvent) => void) {
    const res: Array<(message: InterfaceMessageEvent) => void> | undefined = this.handlers.get(type)
    if (!res) {
      return
    }
    let i = 0
    for (const len = res.length; i < len; i++) {
      if (res[i] === handler) {
        break
      }
    }
    res.splice(i, 1)
    this.handlers.set(type, res)
  }
}

export default WatchEvent
```

---

## 使用範例（預覽）

這個類別本身只是一個基礎工具，真正的使用在 `touch/index.ts` 裡：

```ts
// touch/index.ts 裡會這樣用：
class CropperTouchEvent {
  watcher: WatchEvent

  constructor(element) {
    this.watcher = new WatchEvent()  // 每個 TouchEvent 有自己的 watcher
  }

  on(type, handler) {
    this.watcher.addHandler(type, handler)   // 訂閱
    // ...綁定 DOM 事件
  }

  off(type, handler) {
    this.watcher.removeHandler(type, handler) // 取消訂閱
    // ...解綁 DOM 事件
  }
}

// vue-cropper.vue 裡會這樣用：
cropImg = new CropperTouchEvent(domImg)
cropImg.on('down-to-move', moveImg)    // 訂閱移動事件
cropImg.on('up', reboundImg)           // 訂閱放開事件
```

---

## 小結

`WatchEvent` 是一個輕量的**事件總線（Event Bus）**：

- `addHandler`：訂閱
- `fire`：發送
- `removeHandler`：取消訂閱

它讓「滑鼠/觸控事件的偵測」和「對應的業務邏輯」完全解耦。這種設計讓整個拖曳系統非常容易擴充和測試。

下一篇：[06 — lib/exif.ts + lib/conversion.ts（圖片方向校正）](./06-exif-conversion.md)
