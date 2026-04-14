<LangBlock lang="zh">

# 事件

當前版本實際對外觸發的事件如下。

名稱 | 說明 | 返回值
--- | --- | ---
`img-load` | 圖片加載完成或失敗時觸發 | `{ type: 'success' | 'error', message: string }`
`img-upload` | 拖曳上傳或本地檔案讀取成功時觸發 | `string`
`real-time` | 圖片或截圖框變化時觸發預覽資料 | 預覽物件
`realTime` | `real-time` 的相容別名 | 預覽物件

## `img-load`

```html
<vue-cropper :img="img" @img-load="handleImgLoad" />
```

```ts
const handleImgLoad = (payload) => {
  console.log(payload.type, payload.message)
}
```

成功時：

```ts
{
  type: 'success',
  message: '圖片加載成功'
}
```

失敗時：

```ts
{
  type: 'error',
  message: '圖片加載失敗...'
}
```

## `img-upload`

```html
<vue-cropper :img="img" @img-upload="handleUpload" />
```

```ts
const handleUpload = (url) => {
  img.value = url
}
```

## `real-time`

```html
<vue-cropper :img="img" @real-time="handlePreview" />
```

```ts
const handlePreview = (payload) => {
  console.log(payload.w, payload.h)
  console.log(payload.img.transform)
}
```

返回值結構：

```ts
{
  w: number,
  h: number,
  url: string,
  img: {
    width: string,
    height: string,
    transform: string
  },
  html: string
}
```

## 說明

- 當前版本支援 `real-time` 和 `realTime` 兩種事件名，推薦優先使用 `real-time`。
- `imgMoving`、`cropMoving` 這類舊事件當前仍未開放。
- `imgLoad` 駝峰舊命名仍不作為正式事件，請使用 `img-load`。

</LangBlock>

<LangBlock lang="en">

# Events

The current version emits the following public events.

Name | Description | Payload
--- | --- | ---
`img-load` | Fired when image loading succeeds or fails | `{ type: 'success' | 'error', message: string }`
`img-upload` | Fired after drag upload or local file read succeeds | `string`
`real-time` | Fired when the image or crop box changes and preview data is updated | preview object
`realTime` | Compatibility alias of `real-time` | preview object

## `img-load`

```html
<vue-cropper :img="img" @img-load="handleImgLoad" />
```

```ts
const handleImgLoad = (payload) => {
  console.log(payload.type, payload.message)
}
```

On success:

```ts
{
  type: 'success',
  message: 'Image loaded successfully'
}
```

On failure:

```ts
{
  type: 'error',
  message: 'Image failed to load...'
}
```

## `img-upload`

```html
<vue-cropper :img="img" @img-upload="handleUpload" />
```

```ts
const handleUpload = (url) => {
  img.value = url
}
```

## `real-time`

```html
<vue-cropper :img="img" @real-time="handlePreview" />
```

```ts
const handlePreview = (payload) => {
  console.log(payload.w, payload.h)
  console.log(payload.img.transform)
}
```

Payload shape:

```ts
{
  w: number,
  h: number,
  url: string,
  img: {
    width: string,
    height: string,
    transform: string
  },
  html: string
}
```

## Notes

- Both `real-time` and `realTime` are supported. Prefer `real-time` in new code.
- Old events such as `imgMoving` and `cropMoving` are still not exposed.
- The old camel-case `imgLoad` is not a supported public event. Use `img-load`.

</LangBlock>
