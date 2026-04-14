<LangBlock lang="zh">

# 替換 loading 樣式範例

當前版本支援通過 `loading` 插槽替換預設載入狀態。

### 功能展示

</LangBlock>

<LangBlock lang="en">

# Loading Slot Demo

The current version supports replacing the default loading state through the `loading` slot.

### Demo

</LangBlock>

:::demo
```html
<vue-cropper ref="cropper" :img="img" :crop-layout="{ width: 220, height: 220 }">
  <template #loading>
    <p>{{ labels.loading }}</p>
  </template>
</vue-cropper>
<demo-image-switch v-model="img" />
<crop-export-panel :cropper="cropper" :display-width="220" :display-height="220" />
```

```js
<script setup>
  import { computed, ref } from 'vue'
  import { useLocale } from '../composables/useLocale'
  import { imageList } from '../utils/image'

  const cropper = ref()
  const img = ref('')
  const { isEn } = useLocale()
  const labels = computed(() => isEn.value ? {
    loading: 'Loading...',
    changeImage: 'Change image',
  } : {
    loading: '載入中...',
    changeImage: '切換圖片',
  })
</script>
```
:::

<script setup>
  import { computed, ref } from 'vue'
  import { useLocale } from '../composables/useLocale'
  import { imageList } from '../utils/image'

  const cropper = ref()
  const img = ref('')
  const { isEn } = useLocale()
  const labels = computed(() => isEn.value ? {
    loading: 'Loading...',
    changeImage: 'Change image',
  } : {
    loading: '載入中...',
    changeImage: '切換圖片',
  })
</script>

<style lang="scss" scoped>
  button {
    margin-top: 30px;
  }
</style>
