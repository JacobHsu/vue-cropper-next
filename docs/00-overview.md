# 00 — 全覽：這個專案是什麼

## 目標

這份教學的目標是：從一個**全新的空白資料夾**開始，一步一步建出完整的 `cropper-next-vue` 專案。

你只需要會基本 JavaScript 即可。遇到 TypeScript 或 Vue 3 的新語法，教學會在當下立刻說明。

---

## 這個專案做什麼？

`cropper-next-vue` 是一個 **Vue 3 圖片裁剪元件函式庫**，讓使用者可以：

- 載入圖片（URL 或拖曳上傳）
- 拖曳、縮放、旋轉圖片
- 拉取裁切框選取範圍
- 套用濾鏡（灰階、黑白、老照片）
- 匯出裁切結果（base64 或 Blob）

它發布在 npm 上，其他 Vue 3 專案可以直接安裝使用：

```bash
npm install cropper-next-vue
```

---

## 專案的兩個層次

整個 repo 分為**兩個完全獨立的部分**：

```
vue-cropper-next/
│
├── lib/          ← 函式庫本體（發布到 npm 的內容）
│
└── src/          ← 展示網站（介紹這個函式庫的文件頁面）
```

這兩個部分共用同一份 `package.json`，但有**各自的 Vite 建置設定**：

| 指令 | 執行內容 |
|------|---------|
| `pnpm dev` | 啟動展示網站（開發模式）|
| `pnpm build:lib` | 打包函式庫 → 輸出 `dist/` |
| `pnpm build:docs` | 打包展示網站 → 輸出 `docs-dist/` |

---

## lib/ 的檔案地圖

```
lib/
├── index.ts              ← 函式庫入口（use / install）
├── vue-cropper.vue       ← 主元件（核心）
├── interface.ts          ← 所有 TypeScript 型別定義
├── config.ts             ← 全域常數（阻力係數、回彈時間）
├── common.ts             ← 工具函數集合（圖片載入、Canvas、邊界計算）
├── layoutBox.ts          ← 圖片佈局模式計算（contain / cover / custom）
├── changeImgSize.ts      ← 滑鼠滾輪 / 觸控縮放邏輯
├── watchEvents.ts        ← 事件通知系統（觀察者模式）
├── exif.ts               ← 讀取 JPEG EXIF 方向資訊
├── conversion.ts         ← 根據 EXIF 旋轉校正圖片（Canvas）
├── loading.tsx           ← Loading 元件（使用 JSX 語法）
├── touch/
│   └── index.ts          ← 滑鼠 + 觸控事件統一封裝
├── filter/
│   └── index.ts          ← 圖片濾鏡（灰階、黑白、老照片）
├── style/
│   └── index.scss        ← 元件樣式
└── typings/
    └── index.d.ts        ← 函式庫的 TypeScript 型別聲明檔
```

---

## src/ 的檔案地圖

```
src/
├── main.ts               ← 展示網站入口
├── App.vue               ← 根元件（含側邊欄）
├── pages/                ← 各頁面（大部分是 Markdown 檔）
│   ├── Home.vue
│   ├── Guide.md
│   ├── Props.md
│   ├── DemoBasic.md
│   └── ...（共 14 個頁面）
├── components/           ← 展示網站專用元件
│   ├── Demo.vue          ← 顯示範例 + 原始碼的容器
│   ├── SideBar.vue
│   ├── LocaleSwitch.vue
│   └── ...
├── composables/
│   └── useLocale.ts      ← 多語言切換邏輯
└── assets/               ← 樣式與圖片資源
```

---

## 依賴關係圖

學習順序就是依賴關係的順序——先建底層，再往上疊：

```
interface.ts  config.ts
     ↓              ↓
watchEvents.ts    exif.ts → conversion.ts
     ↓
touch/index.ts
     ↓
         layoutBox.ts
              ↓
         common.ts  ←── changeImgSize.ts
              ↓              ↓
         filter/index.ts   loading.tsx
              ↓
         vue-cropper.vue   ← style/index.scss
              ↓
         index.ts  →  typings/index.d.ts
```

---

## 學習路線

| 篇章 | 檔案 | 重點 |
|------|------|------|
| 01 | — | 環境建置，建立空白專案 |
| 02 | — | Vue 3 快速入門 |
| 03 | lib/ 目錄規劃 | 為何這樣分檔 |
| 04 | interface.ts + config.ts | 型別系統基礎 |
| 05 | watchEvents.ts | 觀察者模式 |
| 06 | exif.ts + conversion.ts | 圖片方向校正 |
| 07 | layoutBox.ts + common.ts 前段 | 圖片佈局計算 |
| 08 | common.ts 後段 | Canvas 操作與邊界動畫 |
| 09 | changeImgSize.ts + touch/ | 縮放與觸控 |
| 10 | filter/index.ts | Canvas 濾鏡原理 |
| 11 | loading.tsx | JSX 元件寫法 |
| 12 | style/index.scss | 樣式設計 |
| 13–17 | vue-cropper.vue（共 5 篇）| 主元件整合 |
| 18 | index.ts + typings/ | 函式庫入口 |
| 19 | 打包設定 | Vite lib 模式 |
| 20–23 | src/（展示網站）| Vue Router + Markdown 渲染 |
