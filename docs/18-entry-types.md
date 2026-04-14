# 18 — lib/index.ts + lib/typings/index.d.ts（入口與型別聲明）

## 函式庫的入口：lib/index.ts

使用者安裝後寫的 `import VueCropper from 'cropper-next-vue'`，最終就是引用這個檔案的導出。

```ts
import VueCropper from './vue-cropper.vue'
import type { VueCropperGlobal } from './typings'
import packageJson from '../package.json'
```

> **TypeScript 穿插說明 — `import packageJson from '../package.json'`**  
> TypeScript 支援直接 `import` JSON 檔案，只要在 `tsconfig.json` 裡設定了 `"resolveJsonModule": true`。  
> 引入後 `packageJson.version` 就是 `"0.1.2"` 這樣的字串，有完整型別支援。

---

### install 函數

```ts
const install = function(app: any) {
  app.component('VueCropper', VueCropper)
}
```

這是 Vue 3 Plugin 的標準格式。

當使用者呼叫 `app.use(VueCropper)` 時，Vue 會尋找物件裡的 `install` 方法並執行。

執行後，整個應用裡所有地方都可以直接用 `<VueCropper />` 而不需要每個元件單獨 import。

**為什麼 `app` 的型別是 `any`？**

Vue 的 `App` 型別定義在 `@vue/runtime-core`，這個套件是 devDependency。函式庫本體為了避免在 runtime 依賴 Vue 的型別，這裡用 `any` 跳過型別檢查。

---

### requestAnimationFrame 的 Polyfill

```ts
if (typeof window !== 'undefined') {
  if (!window.requestAnimationFrame) {
    window.requestAnimationFrame = (fn: FrameRequestCallback) => {
      return window.setTimeout(fn, 17)
    }
  }

  if (!window.cancelAnimationFrame) {
    window.cancelAnimationFrame = id => {
      window.clearTimeout(id)
    }
  }
}
```

**Polyfill**：為不支援某個 API 的環境提供替代實作。

`requestAnimationFrame` 是現代 API，IE 不支援。這段 polyfill 用 `setTimeout(fn, 17)` 模擬（17ms ≈ 60fps）。

**`typeof window !== 'undefined'`**：

在 Node.js 環境（如 SSR 伺服器端渲染）裡，`window` 不存在。用 `typeof` 判斷可以安全地跳過瀏覽器專用的代碼，避免在 Node.js 裡拋出 ReferenceError。

---

### 導出物件

```ts
export const globalCropper: VueCropperGlobal = {
  version: packageJson.version,  // '0.1.2'
  install,
  VueCropper,
}

export { VueCropper }

export default globalCropper
```

設計了**三種使用方式**：

**方式一：全域安裝（最常見）**
```ts
import VueCropper from 'cropper-next-vue'
import 'cropper-next-vue/style.css'

app.use(VueCropper)  // 呼叫 globalCropper.install(app)
// 之後全域可用 <VueCropper />
```

**方式二：按需引用（不全域安裝）**
```ts
import { VueCropper } from 'cropper-next-vue'
import 'cropper-next-vue/style.css'

// 在需要的元件裡手動 import
export default {
  components: { VueCropper }
}
```

**方式三：存取版本號**
```ts
import VueCropper from 'cropper-next-vue'
console.log(VueCropper.version)  // '0.1.2'
```

---

## 完整的 lib/index.ts

```ts
import VueCropper from './vue-cropper.vue'
import type { VueCropperGlobal } from './typings'
import packageJson from '../package.json'

const install = function(app: any) {
  app.component('VueCropper', VueCropper)
}

if (typeof window !== 'undefined') {
  if (!window.requestAnimationFrame) {
    window.requestAnimationFrame = (fn: FrameRequestCallback) => {
      return window.setTimeout(fn, 17)
    }
  }

  if (!window.cancelAnimationFrame) {
    window.cancelAnimationFrame = id => {
      window.clearTimeout(id)
    }
  }
}

export const globalCropper: VueCropperGlobal = {
  version: packageJson.version,
  install,
  VueCropper,
}

export { VueCropper }

export default globalCropper
```

---

## 型別聲明：lib/typings/index.d.ts

### 這個檔案的用途

`.d.ts` 是 TypeScript 的**宣告檔**（Declaration File）。它只包含型別資訊，不包含任何執行邏輯。

當使用者安裝 `cropper-next-vue` 後，TypeScript 會讀取這個檔案，讓他們在使用元件時有型別提示（autocomplete）和錯誤檢查。

```ts
import VueCropper from '../vue-cropper.vue'
import { globalCropper } from '../index'
```

這裡的 import 是在宣告「這些型別來自哪裡」，不是真的在執行期 import。

---

### VueCropperGlobal 介面

```ts
export interface VueCropperGlobal {
  version: string,
  install: (app: any, ...options: any[]) => any,
  VueCropper: typeof VueCropper
}
```

> **TypeScript 穿插說明 — `typeof`（值層面）**  
> 在型別位置使用 `typeof`，可以「取得某個值的型別」。  
> `typeof VueCropper` = 「Vue 元件建構子的型別」（即 VueCropper 這個元件的型別）。  
> 這樣使用者知道 `VueCropperGlobal.VueCropper` 是一個 Vue 元件，不是普通函數或物件。

> **TypeScript 穿插說明 — `...options: any[]`**  
> `...` 是剩餘參數（Rest Parameters），表示「可以接受任意數量的額外參數」。  
> `options: any[]` 說明這些額外參數是 `any` 型別的陣列。  
> 這符合 Vue 3 Plugin 的 `install(app, options?)` 簽名。

---

### 導出

```ts
export {
  VueCropper
}

export default globalCropper
```

讓使用者可以：

```ts
import type { VueCropperGlobal } from 'cropper-next-vue'
// 使用型別寫型別標注
const myPlugin: VueCropperGlobal = ...
```

---

## 完整的 lib/typings/index.d.ts

```ts
import VueCropper from '../vue-cropper.vue'
import { globalCropper } from '../index'

export interface VueCropperGlobal {
  version: string,
  install: (app: any, ...options: any[]) => any,
  VueCropper: typeof VueCropper
}

export {
  VueCropper
}

export default globalCropper
```

---

## 小結

| 檔案 | 角色 |
|------|------|
| `lib/index.ts` | 函式庫的執行期入口：Plugin install + 導出 |
| `lib/typings/index.d.ts` | TypeScript 的型別入口：讓使用者有型別提示 |

這兩個檔案合在一起，完成了函式庫對外的完整介面。

下一篇：[19 — 打包設定（vite.lib.config.ts + tsconfig.lib.json）](./19-build-config.md)
