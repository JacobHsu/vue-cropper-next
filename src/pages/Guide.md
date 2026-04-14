<LangBlock lang="zh">

# 快速開始

`cropper-next-vue` 當前更適合這樣理解：

- 固定截圖框
- 圖片可拖曳、縮放、旋轉
- 支援邊界限制
- 支援即時預覽和高分屏導出

### 安裝

```bash
npm install cropper-next-vue
```

```bash
yarn add cropper-next-vue
```

### 使用

`Vue 3` 元件內引入

```ts
import 'cropper-next-vue/style.css'
import { VueCropper } from 'cropper-next-vue'
```

`Vue 3` 全域引入

```ts
import { createApp } from 'vue'
import App from './App.vue'
import CropperNextVue from 'cropper-next-vue'
import 'cropper-next-vue/style.css'

const app = createApp(App)
app.use(CropperNextVue)
app.mount('#app')
```

### 本地開發命令

```bash
# 文件站開發
pnpm run dev

# 構建 npm 套件
pnpm run build:lib

# 構建文件站
pnpm run build:docs
```

### 推薦閱讀路徑

如果你第一次接觸這個庫，建議按這個順序看文件：

1. [基礎範例](#/demo-basic)：先跑通最小裁剪和導出。
2. [導出能力](#/demo-export)：理解 `outputType`、`outputSize`、`full`、`getCropBlob`。
3. [邊界控制](#/demo-img)：理解 `centerBox` 和 `centerWrapper`。
4. [旋轉控制](#/demo-rotate)：理解旋轉和邊界約束的組合。
5. [即時預覽](#/demo-realtime)：理解 `real-time`、實例旋轉方法和聯動預覽。

### 當前能力邊界

當前版本走的是「固定截圖框」路線，因此沒有舊版那套可縮放裁剪框、八點拖曳和固定比例裁剪框 API。  
如果你的需求重點是旋轉後的裁剪、邊界控制、導出品質和元件整合，這個版本更合適。

</LangBlock>

<LangBlock lang="en">

# Guide

`cropper-next-vue` is best understood as:

- a fixed crop-box component
- draggable, scalable, and rotatable image editing
- boundary control support
- realtime preview and high-DPI export support

### Install

```bash
npm install cropper-next-vue
```

```bash
yarn add cropper-next-vue
```

### Usage

Import inside a Vue 3 component:

```ts
import 'cropper-next-vue/style.css'
import { VueCropper } from 'cropper-next-vue'
```

Global registration in Vue 3:

```ts
import { createApp } from 'vue'
import App from './App.vue'
import CropperNextVue from 'cropper-next-vue'
import 'cropper-next-vue/style.css'

const app = createApp(App)
app.use(CropperNextVue)
app.mount('#app')
```

### Local development commands

```bash
# docs dev server
pnpm run dev

# build npm package
pnpm run build:lib

# build docs site
pnpm run build:docs
```

### Recommended reading order

If this is your first time using the library, this sequence works best:

1. [Basic Demo](#/demo-basic): get the minimal crop and export flow working.
2. [Export](#/demo-export): understand `outputType`, `outputSize`, `full`, and `getCropBlob`.
3. [Boundary](#/demo-img): understand `centerBox` and `centerWrapper`.
4. [Rotation](#/demo-rotate): understand rotation plus boundary constraints.
5. [Realtime Preview](#/demo-realtime): understand `real-time`, rotation methods, and linked preview.

### Current scope

This version follows a fixed crop-box direction, so it does not include the old resizable crop-box APIs, eight-point resize handles, or fixed-ratio crop-box workflow.  
If your priority is rotated cropping, boundary control, export quality, and Vue integration, this version is a better fit.

</LangBlock>
