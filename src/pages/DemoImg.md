<LangBlock lang="zh">

# 邊界控制

這頁用來理解兩種最核心的限制策略：

- `centerBox`：圖片必須完整包住截圖框
- `centerWrapper`：圖片必須留在外層容器內

建議你分別拖曳、縮放，再對比兩組行為差異。

### 圖片限制範例

</LangBlock>

<LangBlock lang="en">

# Boundary Control

This page explains the two core boundary strategies:

- `centerBox`: the image must fully cover the crop box
- `centerWrapper`: the image must stay inside the wrapper

Try dragging and zooming both demos to compare the behavior.

### Boundary examples

</LangBlock>

:::demo
```html
<p class="title">{{ labels.centerBoxTitle }}</p>
<vue-cropper 
  center-box
  :center-box-delay="150"
  ref="cropper1"
  :img="img"
  :crop-layout="{ width: 220, height: 220 }"
>
</vue-cropper>
<demo-image-switch v-model="img" />
<p class="desc">{{ labels.centerBoxDesc }}</p>
<crop-export-panel :cropper="cropper1" :display-width="220" :display-height="220" />

<p class="title">{{ labels.centerWrapperTitle }}</p>
<vue-cropper 
  center-wrapper
  :center-wrapper-delay="150"
  ref="cropper2"
  :img="img"
  :crop-layout="{ width: 220, height: 220 }"
>
</vue-cropper>
<p class="desc">{{ labels.centerWrapperDesc }}</p>
<crop-export-panel :cropper="cropper2" :display-width="220" :display-height="220" />
```

```js
<script setup>
  import { computed, ref } from 'vue'
  import { useLocale } from '../composables/useLocale'

  const cropper1 = ref()
  const cropper2 = ref()
  const img = ref('')
  const { isEn } = useLocale()
  const labels = computed(() => isEn.value ? {
    centerBoxTitle: 'Keep image covering crop box',
    centerBoxDesc: 'Useful when the final output must fully cover the crop area, such as avatar cropping.',
    centerWrapperTitle: 'Keep image inside wrapper',
    centerWrapperDesc: 'Useful for editing flows where the image must always remain inside the workspace.',
  } : {
    centerBoxTitle: '圖片限制截圖框內',
    centerBoxDesc: '適合最終必須鋪滿裁剪區域的場景，比如頭像裁剪。',
    centerWrapperTitle: '圖片限制容器內',
    centerWrapperDesc: '適合希望圖片始終留在工作區內的編輯場景。',
  })
</script>
```
:::

<script setup>
  import { computed, ref } from 'vue'
  import { useLocale } from '../composables/useLocale'

  const cropper1 = ref()
  const cropper2 = ref()
  const img = ref('')
  const { isEn } = useLocale()
  const labels = computed(() => isEn.value ? {
    centerBoxTitle: 'Keep image covering crop box',
    centerBoxDesc: 'Useful when the final output must fully cover the crop area, such as avatar cropping.',
    centerWrapperTitle: 'Keep image inside wrapper',
    centerWrapperDesc: 'Useful for editing flows where the image must always remain inside the workspace.',
  } : {
    centerBoxTitle: '圖片限制截圖框內',
    centerBoxDesc: '適合最終必須鋪滿裁剪區域的場景，比如頭像裁剪。',
    centerWrapperTitle: '圖片限制容器內',
    centerWrapperDesc: '適合希望圖片始終留在工作區內的編輯場景。',
  })
</script>

<style lang="scss" scoped>
  .title {
    margin: 20px 0 8px;
    font-weight: 600;
  }

  .desc {
    margin: 12px 0 0;
    color: #666;
  }
</style>
