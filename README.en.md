# cropper-next-vue

![license](https://img.shields.io/badge/license-ISC-blue.svg)
![node](https://img.shields.io/badge/node-22.x-339933.svg)
![coverage](https://img.shields.io/badge/coverage-%E2%89%A570%25-brightgreen.svg)
![tests](https://img.shields.io/badge/tests-passing-brightgreen.svg)

[繁體中文](./README.md) | English

`cropper-next-vue` is a standalone Vue 3 image cropper focused on:

- boundary handling after rotation
- keeping the image inside the crop box or wrapper
- high-DPI export
- realtime preview
- standalone npm package output
- standalone docs site output

Live preview:

- [https://cropper-next-vue.vercel.app/](https://cropper-next-vue.vercel.app/)

## Install

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

## Usage

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

## Local development

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

## Quality gates

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

## Build outputs

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

## Open source collaboration

- License: `ISC`
- Required Node version: `22.x`
- Recommended pre-commit check: `pnpm run check`
- Contribution guide: [CONTRIBUTING.md](./CONTRIBUTING.md)
- Code of conduct: [CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md)
