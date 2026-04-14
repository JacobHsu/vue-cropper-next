<LangBlock lang="zh">

# 基礎範例

這個頁面建議先完成 3 個動作：

1. 拖曳圖片，確認截圖區域跟隨變化。
2. 滑鼠滾輪縮放，觀察截圖結果。
3. 點擊導出，確認已經能拿到 base64 結果。

### 最小可用範例

</LangBlock>

<LangBlock lang="en">

# Basic Demo

Try these three actions first:

1. Drag the image and confirm the crop area follows correctly.
2. Use the mouse wheel to zoom.
3. Export once and confirm you receive a base64 result.

### Minimal working example

</LangBlock>

:::demo
```html
<vue-cropper
  ref="cropper"
  :img="img"
  :crop-layout="{ width: 220, height: 220 }"
>
</vue-cropper>
<demo-image-switch v-model="img" />
<section class="tips">
  <p>{{ labels.tipFlow }}</p>
  <p>{{ labels.tipRetina }}</p>
</section>
<crop-export-panel :cropper="cropper" :display-width="220" :display-height="220" />
```

```js
<script setup>
  import { computed, ref } from 'vue'
  import { useLocale } from '../composables/useLocale'

  const cropper = ref()
  const img = ref('')
  const { isEn } = useLocale()
  const labels = computed(() => isEn.value ? {
    tipFlow: 'Suggested flow: drag first, zoom with the wheel, then export.',
    tipRetina: 'This demo uses the default full=true, so export is high-DPI friendly by default.',
  } : {
    tipFlow: '操作建議：拖曳圖片後再用滾輪縮放，然後點擊導出。',
    tipRetina: '這個 demo 使用預設 full=true，導出結果預設更適合高分屏。',
  })
</script>
```
:::

<script setup>
  import { computed, ref } from 'vue'
  import { useLocale } from '../composables/useLocale'

  const cropper = ref()
  const img = ref('')
  const { isEn } = useLocale()
  const labels = computed(() => isEn.value ? {
    tipFlow: 'Suggested flow: drag first, zoom with the wheel, then export.',
    tipRetina: 'This demo uses the default full=true, so export is high-DPI friendly by default.',
  } : {
    tipFlow: '操作建議：拖曳圖片後再用滾輪縮放，然後點擊導出。',
    tipRetina: '這個 demo 使用預設 full=true，導出結果預設更適合高分屏。',
  })
</script>

<style lang="scss" scoped>
  .tips {
    margin-top: 20px;
    color: #666;
    line-height: 1.8;
  }

  p {
    margin: 0;
  }
</style>
