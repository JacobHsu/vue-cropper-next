<LangBlock lang="zh">

# 方法

當前版本通過元件 `ref` 暴露的方法較少，主要用於獲取裁剪結果。

## 獲取實例

```ts
const cropper = ref()
```

```html
<vue-cropper ref=”cropper” :img=”img” />
```

## 方法列表

方法 | 說明
--- | ---
`cropper.value.getCropData(type?)` | 獲取裁剪結果，返回 `Promise<string>`
`cropper.value.getCropBlob()` | 獲取裁剪結果，返回 `Promise<Blob>`
`cropper.value.rotateLeft()` | 向左旋轉 `90deg`
`cropper.value.rotateRight()` | 向右旋轉 `90deg`
`cropper.value.rotateClear()` | 清空旋轉角度，恢復為 `0deg`

## 參數說明

`getCropData(type?)`

- 預設返回 base64 資料
- 當前實作會根據元件的 `outputType` 輸出對應格式
- `type` 參數當前主要用於相容呼叫方式，推薦直接使用預設值

`getCropBlob()`

- 返回 `Blob`
- 更適合直接上傳到伺服端或和 `FormData` 搭配使用

## 範例

```ts
cropper.value.getCropData().then((data) => {
  console.log(data)
})
```

```ts
cropper.value.getCropBlob().then((blob) => {
  const formData = new FormData()
  formData.append('file', blob, 'crop.png')
})
```

## 說明

當前版本仍然沒有舊版的 `startCrop`、`stopCrop`、`clearCrop`、`changeScale`、`getImgAxis`、`getCropAxis`、`goAutoCrop`。這些屬於舊版「可變裁剪框」路線，和當前實作不一致。

</LangBlock>

<LangBlock lang="en">

# Methods

The current version exposes a small set of instance methods through component `ref`, mainly focused on export and rotation control.

## Get the instance

```ts
const cropper = ref()
```

```html
<vue-cropper ref="cropper" :img="img" />
```

## Method list

Method | Description
--- | ---
`cropper.value.getCropData(type?)` | Get crop result as `Promise<string>`
`cropper.value.getCropBlob()` | Get crop result as `Promise<Blob>`
`cropper.value.rotateLeft()` | Rotate left by `90deg`
`cropper.value.rotateRight()` | Rotate right by `90deg`
`cropper.value.rotateClear()` | Reset rotation back to `0deg`

## Details

`getCropData(type?)`

- returns base64 by default
- uses the component `outputType` as the export format
- the `type` parameter is kept mainly for compatibility, and the default is recommended

`getCropBlob()`

- returns a `Blob`
- better suited for uploads and `FormData`

## Example

```ts
cropper.value.getCropData().then((data) => {
  console.log(data)
})
```

```ts
cropper.value.getCropBlob().then((blob) => {
  const formData = new FormData()
  formData.append('file', blob, 'crop.png')
})
```

## Notes

The current version still does not include old APIs such as `startCrop`, `stopCrop`, `clearCrop`, `changeScale`, `getImgAxis`, `getCropAxis`, or `goAutoCrop`. Those belonged to the old resizable crop-box direction.

</LangBlock>
