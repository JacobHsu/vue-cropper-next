# cropper-next-vue

![license](https://img.shields.io/badge/license-ISC-blue.svg)
![node](https://img.shields.io/badge/node-22.x-339933.svg)
![coverage](https://img.shields.io/badge/coverage-%E2%89%A570%25-brightgreen.svg)
![tests](https://img.shields.io/badge/tests-passing-brightgreen.svg)

[中文](#中文) | [English](#english)

## 中文

`cropper-next-vue` 是一個獨立發佈的 Vue 3 圖片裁剪庫，重點處理這些能力：

- 圖片旋轉後的邊界判斷
- 圖片限制在截圖框內或容器內
- 高分屏導出
- 即時預覽
- 獨立的 npm 套件構建
- 獨立的文件站構建

線上預覽：

- [https://cropper-next-vue.vercel.app/](https://cropper-next-vue.vercel.app/)

### 安裝

推薦使用 pnpm：

```bash
pnpm add cropper-next-vue
```

```bash
npm install cropper-next-vue
```

```bash
yarn add cropper-next-vue
```

### 使用

元件內引入：

```ts
import 'cropper-next-vue/style.css'
import { VueCropper } from 'cropper-next-vue'
```

全域註冊：

```ts
import { createApp } from 'vue'
import App from './App.vue'
import CropperNextVue from 'cropper-next-vue'
import 'cropper-next-vue/style.css'

const app = createApp(App)
app.use(CropperNextVue)
app.mount('#app')
```

基礎範例：

```vue
<template>
  <VueCropper
    :img="img"
    :crop-layout="{ width: 200, height: 200 }"
    :center-box="true"
  />
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { VueCropper } from 'cropper-next-vue'
import 'cropper-next-vue/style.css'

const img = ref('https://example.com/demo.jpg')
</script>
```

### 本地開發

如果你未啟用 corepack（Node 16+），可以先執行 `corepack enable`。

```bash
pnpm install

# 文件站開發
pnpm run dev

# 只構建 npm 套件
pnpm run build:lib

# 只構建文件站
pnpm run build:docs

# 同時構建 npm 套件和文件站
pnpm run build
```

### 品質門檻

- `pnpm run typecheck`
- `pnpm run test:coverage`
- `pnpm run build:lib`
- `pnpm run check`

其中 `pnpm run check` 會依次執行：typecheck → test:coverage → build:lib。

覆蓋率閾值定義在 [vitest.config.ts](./vitest.config.ts)：

- `lines >= 70`
- `functions >= 70`
- `branches >= 60`
- `statements >= 70`

### 構建輸出

- npm 套件輸出到 `dist/`
- 文件站輸出到 `docs-dist/`

發佈前建議執行：

```bash
pnpm run build:lib
pnpm pack --pack-destination /tmp
```

發佈 npm 可直接使用：

```bash
pnpm run release:npm -- patch
pnpm run release:npm -- minor
pnpm run release:npm -- 0.2.0
pnpm run release:npm -- patch --tag next
```

這個腳本會依次執行：

- 更新 `package.json` 版本號
- 執行 `pnpm run check`
- 重新構建 lib 產物
- 發佈到 npm

預設要求 git 工作區乾淨，並且當前機器已完成 `npm login`。

### 開源協作

- 授權條款：`ISC`
- Node 版本要求：`22.x`
- 提交前建議執行：`pnpm run check`
- 貢獻說明見 [CONTRIBUTING.md](./CONTRIBUTING.md)
- 行為準則見 [CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md)

## English

`cropper-next-vue` is a standalone Vue 3 image cropper focused on:

- boundary handling after rotation
- keeping the image inside the crop box or wrapper
- high-DPI export
- realtime preview
- standalone npm package output
- standalone docs site output

Live preview:

- [https://cropper-next-vue.vercel.app/](https://cropper-next-vue.vercel.app/)

### Install

Recommended with pnpm:

```bash
pnpm add cropper-next-vue
```

```bash
npm install cropper-next-vue
```

```bash
yarn add cropper-next-vue
```

### Usage

Import inside a component:

```ts
import 'cropper-next-vue/style.css'
import { VueCropper } from 'cropper-next-vue'
```

Global registration:

```ts
import { createApp } from 'vue'
import App from './App.vue'
import CropperNextVue from 'cropper-next-vue'
import 'cropper-next-vue/style.css'

const app = createApp(App)
app.use(CropperNextVue)
app.mount('#app')
```

Basic example:

```vue
<template>
  <VueCropper
    :img="img"
    :crop-layout="{ width: 200, height: 200 }"
    :center-box="true"
  />
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { VueCropper } from 'cropper-next-vue'
import 'cropper-next-vue/style.css'

const img = ref('https://example.com/demo.jpg')
</script>
```

### Local development

If corepack is not enabled (Node 16+), run `corepack enable` first.

```bash
pnpm install

# docs dev server
pnpm run dev

# build npm package only
pnpm run build:lib

# build docs site only
pnpm run build:docs

# build both package and docs
pnpm run build
```

### Quality gates

- `pnpm run typecheck`
- `pnpm run test:coverage`
- `pnpm run build:lib`
- `pnpm run check`

`pnpm run check` runs: typecheck → test:coverage → build:lib.

Coverage thresholds are defined in [vitest.config.ts](./vitest.config.ts):

- `lines >= 70`
- `functions >= 70`
- `branches >= 60`
- `statements >= 70`

### Build outputs

- npm package output goes to `dist/`
- docs site output goes to `docs-dist/`

Recommended before publishing:

```bash
pnpm run build:lib
pnpm pack --pack-destination /tmp
```

Release to npm:

```bash
pnpm run release:npm -- patch
pnpm run release:npm -- minor
pnpm run release:npm -- 0.2.0
pnpm run release:npm -- patch --tag next
```

The release script will:

- update the version in `package.json`
- run `pnpm run check`
- rebuild the library output
- publish the package to npm

It requires a clean git working tree and a valid `npm login` session by default.

### Open source collaboration

- License: `ISC`
- Required Node version: `22.x`
- Recommended pre-commit check: `pnpm run check`
- Contribution guide: [CONTRIBUTING.md](./CONTRIBUTING.md)
- Code of conduct: [CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md)
