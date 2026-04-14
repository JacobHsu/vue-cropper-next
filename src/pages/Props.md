<LangBlock lang="zh">

# 參數

當前版本暴露的元件參數以 [lib/vue-cropper.vue](/Users/bytedance/projects/cropper-next/lib/vue-cropper.vue) 為準。

名稱 | 功能 | 預設值 | 類型 / 可選值
--- | --- | --- | ---
img | 圖片位址 | `''` | `string`
wrapper | 外層容器寬高 | `{ width: 300, height: 300 }` | `{ width: number \| string; height: number \| string }`
cropLayout | 截圖框大小 | `{ width: 200, height: 200 }` | `{ width: number \| string; height: number \| string }`
color | 主題色預留欄位 | `'#fff'` | `string`
filter | 圖片濾鏡函數 | `null` | `(canvas) => canvas`
outputType | 輸出圖片格式 | `'png'` | `jpeg`、`png`、`webp`
outputSize | 輸出圖片品質 | `1` | `number`，建議 `0-1`
full | 是否按高分屏方式導出 | `true` | `boolean`
original | 按原圖比例導出（跟隨當前縮放倍數放大導出像素） | `false` | `boolean`
maxSideLength | 限制導出圖片最長邊像素（`0` 表示不限制） | `3000` | `number`
mode | 圖片初始佈局方式 | `'cover'` | `contain`、`cover`、`original`、`100px`、`100%`、`auto 100px` 等
cropColor | 截圖框描邊顏色 | `'#fff'` | `string`
defaultRotate | 預設旋轉角度 | `0` | `number`
centerBox | 圖片是否限制在截圖框內 | `false` | `boolean`
centerWrapper | 圖片是否限制在容器內 | `false` | `boolean`
centerBoxDelay | 圖片限制截圖框內時的回彈時長 | `100` | `number`
centerWrapperDelay | 圖片限制容器內時的回彈時長 | `100` | `number`

## 說明

- `centerBox` 和 `centerWrapper` 可以分別控制兩種邊界限制策略。
- 當圖片發生旋轉後，邊界限制會重新校驗。
- `filter` 接收一個 `HTMLCanvasElement`，返回處理後的 `HTMLCanvasElement`。
- `outputSize` 會影響 `jpeg / webp` 等格式的壓縮品質（取值範圍 `0-1`，預設 `1`）。
  - `1`：最高畫質，檔案最大，適合高清裁剪圖、列印或高分屏展示。
  - `0.9`：畫質接近原圖，但檔案明顯更小，適合網頁預覽或社群分享。
  - `0.8` 及以下：壓縮更大，檔案更小，適合批量導出或對畫質要求不高的場景。
  - 一般網頁/社群分享建議 `0.9`，追求最清晰視覺效果可保持 `1`。
- `full` 預設開啟，導出時會按當前裝置像素比生成更適合高分屏的結果。
- `original` 影響導出像素：開啟後，會把導出解析度按當前縮放倍數放大，以盡量貼近原圖解析度（仍會受 `maxSideLength` 限制）。
- `maxSideLength` 用於保護導出效能，預設把最長邊壓到 `3000` 以內；傳 `0` 可關閉該限制。
- `wrapper` 和 `cropLayout` 現在都支援傳 `number` 或 `string`，例如 `300`、`'300px'`、`'60%'`。
- 當前版本沒有舊版 `autoCrop`、`fixed`、`canMoveBox`、`enlarge`、`maxImgSize` 等參數，這些屬於舊實作，不再適用。

</LangBlock>

<LangBlock lang="en">

# Props

The current public props are defined by [lib/vue-cropper.vue](/Users/bytedance/projects/cropper-next/lib/vue-cropper.vue).

Name | Purpose | Default | Type / Allowed values
--- | --- | --- | ---
img | Image source | `''` | `string`
wrapper | Outer container size | `{ width: 300, height: 300 }` | `{ width: number \| string; height: number \| string }`
cropLayout | Crop-box size | `{ width: 200, height: 200 }` | `{ width: number \| string; height: number \| string }`
color | Reserved theme color field | `'#fff'` | `string`
filter | Image filter callback | `null` | `(canvas) => canvas`
outputType | Export image format | `'png'` | `jpeg`, `png`, `webp`
outputSize | Export quality | `1` | `number`, recommended `0-1`
full | Export for high-DPI output | `true` | `boolean`
original | Export using the original pixel ratio (scale export pixels up by the current zoom) | `false` | `boolean`
maxSideLength | Clamp export max edge size (`0` disables clamping) | `3000` | `number`
mode | Initial image layout mode | `'cover'` | `contain`, `cover`, `original`, `100px`, `100%`, `auto 100px`, etc.
cropColor | Crop-box outline color | `'#fff'` | `string`
defaultRotate | Initial rotation angle | `0` | `number`
centerBox | Keep image covering the crop box | `false` | `boolean`
centerWrapper | Keep image inside the wrapper | `false` | `boolean`
centerBoxDelay | Rebound duration for `centerBox` | `100` | `number`
centerWrapperDelay | Rebound duration for `centerWrapper` | `100` | `number`

## Notes

- `centerBox` and `centerWrapper` control two different boundary strategies.
- Boundary checks are recalculated after rotation.
- `filter` receives an `HTMLCanvasElement` and should return a processed `HTMLCanvasElement`.
- `outputSize` affects compressed formats such as `jpeg` and `webp` (range `0-1`, default `1`).
  - `1`: highest quality, largest file, good for crisp exports and retina display.
  - `0.9`: near-original quality with a noticeably smaller file, good for web preview/sharing.
  - `0.8` or lower: more compression and smaller files, good for batch export.
  - Recommended: `0.9` for most web/sharing, keep `1` for maximum clarity.
- `full` is enabled by default, so exports use the current device pixel ratio for sharper high-DPI output.
- `original` affects export pixel size: when enabled, export resolution scales up by the current zoom level (still clamped by `maxSideLength`).
- `maxSideLength` protects export performance by clamping the longest edge to `3000` by default; pass `0` to disable.
- `wrapper` and `cropLayout` now both accept `number` or `string`, such as `300`, `'300px'`, or `'60%'`.
- The current version does not include old props such as `autoCrop`, `fixed`, `canMoveBox`, `enlarge`, or `maxImgSize`.

</LangBlock>
