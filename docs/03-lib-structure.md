# 03 — lib/ 目錄規劃與設計思路

## 為什麼要單獨規劃 lib/ 目錄？

這個專案不只是一個「給自己用的網頁」，它要**打包成一個 npm 套件**讓別人安裝。

這代表 lib/ 的程式碼需要：
1. **不依賴 src/**（函式庫本體不能依賴展示網站的程式碼）
2. **有獨立的打包設定**（`vite.lib.config.ts`，區別於展示網站的 `vite.config.ts`）
3. **輸出型別宣告檔**（讓使用 TypeScript 的人有 autocomplete）
4. **體積盡量小**（外部依賴越少越好，Vue 讓使用者自己提供）

---

## 分檔原則：一個檔案一個職責

好的函式庫程式碼有一個特徵：**每個檔案只做一件事**。

|檔案 | 職責 |
|-----|------|
| `interface.ts` | 只放型別定義，沒有任何邏輯 |
| `config.ts` | 只放常數，沒有任何邏輯 |
| `watchEvents.ts` | 只處理事件的訂閱與發送 |
| `exif.ts` | 只讀取 EXIF 資料 |
| `conversion.ts` | 只做圖片旋轉校正 |
| `layoutBox.ts` | 只計算圖片的初始縮放比例 |
| `common.ts` | 工具函數集合（圖片載入、Canvas、邊界計算）|
| `changeImgSize.ts` | 只處理滑鼠滾輪 / 觸控縮放 |
| `loading.tsx` | 只是一個 Loading 元件 |
| `filter/index.ts` | 只放濾鏡函數 |
| `touch/index.ts` | 只封裝滑鼠 + 觸控事件 |
| `style/index.scss` | 只放樣式 |
| `vue-cropper.vue` | 主元件，整合上面所有模組 |
| `index.ts` | 只做導出，讓外部 import 有統一入口 |

這種設計讓每個模組都**可以獨立測試**，也讓 `vue-cropper.vue` 主元件只專注在邏輯整合，不被雜務干擾。

---

## 依賴方向：只能往下，不能往上

```
interface.ts    config.ts         ← 底層：不依賴任何其他 lib 檔案
     ↓               ↓
watchEvents.ts    exif.ts → conversion.ts
     ↓
touch/index.ts
     ↓
         layoutBox.ts
              ↓
         common.ts  ←── changeImgSize.ts
              ↓
         filter/index.ts   loading.tsx
              ↓
         vue-cropper.vue   ← style/index.scss
              ↓
         index.ts  →  typings/index.d.ts
```

規則：**底層檔案不可以引用上層檔案**。例如 `watchEvents.ts` 只引用 `interface.ts`，不可以引用 `vue-cropper.vue`。
反過來則可以：`vue-cropper.vue` 可以引用 `watchEvents.ts`、`common.ts`、`touch/index.ts` 等底層模組，因為它本來就是上層整合點。真正會引用 `vue-cropper.vue` 的，通常是更外層的入口檔，例如這個專案的 `lib/index.ts`。

違反這個原則會產生**循環依賴**，讓打包工具無法確定模組載入順序，造成難以追蹤的 bug。

---

## 型別優先：先定義形狀，再寫邏輯

第一個要寫的檔案永遠是 `interface.ts`。

原因：TypeScript 的型別定義像是**合約**，先把「這個函數接受什麼、回傳什麼」寫清楚，後續實作時 TypeScript 就會提醒你有沒有違反合約。

這個習慣在多人協作時特別重要——先定型別，大家就知道「要實作什麼」，不用等到程式碼寫完才發現介面不相容。

---

## 為什麼用 .tsx（JSX）而不是 .vue？

`loading.tsx` 使用 JSX 語法，而不是 `.vue` 的 `<template>` 語法。

`.vue` 適合：複雜的 template 邏輯、需要 `scoped` 樣式、需要複雜的生命週期。

`.tsx` 適合：**結構非常簡單**的元件，邏輯和 JSX 放在一起更清晰。

Loading 元件只有「顯示/隱藏」加上一個 SVG，用 JSX 更精簡。

---

## 為什麼 touch/ 和 filter/ 放在子目錄？

兩個原因：

1. **可擴充性**：未來 `filter/` 可能新增更多濾鏡（如銳化、模糊），放在獨立目錄方便管理。
2. **清楚的分組**：`touch/` 裡的東西都是「觸控相關」，`filter/` 裡的東西都是「濾鏡相關」，看目錄名稱就知道用途。

這種分組方式叫做**按功能（feature）組織**，是大型專案的常見慣例。

---

## 先建立空檔案

在開始寫內容之前，先建立所有需要的空檔案：

```bash
# 核心工具
touch lib/interface.ts
touch lib/config.ts
touch lib/watchEvents.ts
touch lib/exif.ts
touch lib/conversion.ts
touch lib/layoutBox.ts
touch lib/common.ts
touch lib/changeImgSize.ts
touch lib/loading.tsx

# 子目錄
touch lib/touch/index.ts
touch lib/filter/index.ts
touch lib/style/index.scss

# 主元件與入口
touch lib/vue-cropper.vue
touch lib/index.ts
touch lib/typings/index.d.ts
```

完成後 lib/ 目錄長這樣：

```
lib/
├── interface.ts
├── config.ts
├── watchEvents.ts
├── exif.ts
├── conversion.ts
├── layoutBox.ts
├── common.ts
├── changeImgSize.ts
├── loading.tsx
├── vue-cropper.vue
├── index.ts
├── touch/
│   └── index.ts
├── filter/
│   └── index.ts
├── style/
│   └── index.scss
└── typings/
    └── index.d.ts
```

接下來就按照依賴順序，從底層開始逐一填入內容。

下一篇：[04 — interface.ts + config.ts](./04-interface-config.md)
