# Contributing

感謝你為 `cropper-next-vue` 做出的貢獻。

## 開發要求

- Node.js `>=18`
- npm `>=9`

## 本地流程

```bash
npm install
npm run dev
```

常用檢查命令：

```bash
npm run typecheck
npm run test:coverage
npm run build:lib
npm run build:docs
npm run check
```

## 提交要求

- 保持套件構建和文件構建都可用
- 新增行為優先補測試
- 不提交無關構建產物或臨時偵錯程式碼
- 發佈前至少執行一次 `npm run check`

## 變更範圍

歡迎這些方向的改進：

- 裁剪幾何和邊界控制
- Vue 3 相容性和類型品質
- 文件、範例和測試覆蓋率
- 發佈流程和工程品質
