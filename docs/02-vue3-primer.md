# 02 — Vue 3 快速入門（本專案用到的部分）

> 這篇只教本專案**實際用到**的 Vue 3 語法。你不需要讀完整份 Vue 文件，看完這篇就夠用。

---

## Vue 是什麼？

Vue 是一個讓 JS 能「驅動 HTML」的框架。

傳統 JS：你要自己找到 DOM 元素、手動更新它。  
Vue：你只管「資料」，畫面自動跟著更新。

```html
<!-- 傳統 JS -->
<p id="name"></p>
<script>
  document.getElementById('name').textContent = '小明'
</script>

<!-- Vue 3 -->
<template>
  <p>{{ name }}</p>
</template>
<script setup>
  const name = '小明'
</script>
```

---

## 單檔元件（SFC）

Vue 的檔案副檔名是 `.vue`，一個檔案包含三個區塊：

單檔元件（SFC, Single File Component）是指一個 Vue 元件用一個 `.vue` 檔案完整描述，把畫面結構、元件邏輯、元件樣式放在同一個檔案裡。

```vue
<script setup lang="ts">
// 邏輯（TypeScript）
</script>

<template>
  <!-- HTML 結構 -->
</template>

<style scoped>
/* 樣式（只作用在這個元件）*/
</style>
```

**`<script setup>`** 是 Vue 3 的語法糖，裡面宣告的變數可以直接在 template 使用。  
**`lang="ts"`** 表示用 TypeScript 撰寫。

---

## ref：讓基本型別變成響應式

Vue 的核心概念：只有被「包裝」過的資料，畫面才會自動更新。

```ts
import { ref } from 'vue'

const count = ref(0)         // 建立一個響應式數字
count.value++                // 在 JS 裡要用 .value 取值
```

在 template 裡不需要寫 `.value`，Vue 自動幫你取：

```html
<p>{{ count }}</p>           <!-- 自動顯示 count.value -->
<button @click="count++">+1</button>
```

> **TypeScript 穿插說明 — 型別推斷**  
> `ref(0)` 的型別會被 TypeScript 自動推斷為 `Ref<number>`。  
> 如果需要明確指定，可以寫 `ref<string>('')`。

---

## reactive：讓物件變成響應式

```ts
import { reactive } from 'vue'

const state = reactive({
  x: 0,
  y: 0,
  scale: 1,
})

state.x = 100  // 直接修改屬性，不需要 .value
```

`reactive` 適合**一組相關的狀態**放在一起管理。  
本專案的 `LayoutContainer` 就是用 `reactive` 把所有佈局狀態集中管理。

---

## computed：自動計算的衍生值

```ts
import { ref, computed } from 'vue'

const width = ref(300)
const height = ref(200)

const area = computed(() => width.value * height.value)
// area.value 自動是 60000
// 當 width 或 height 改變，area 也自動重新計算
```

`computed` 有快取：只有依賴的值改變時才重新計算，否則直接回傳上次結果。

---

## watch：監聽資料變化

```ts
import { ref, watch } from 'vue'

const imgUrl = ref('')

watch(imgUrl, (newVal, oldVal) => {
  // 當 imgUrl 改變時執行
  console.log('圖片 URL 變了：', newVal)
})
```

監聽深層物件（物件內的屬性改變）：

```ts
watch(wrapperStyle, () => {
  // 物件的任何屬性改變都會觸發
}, { deep: true })
```

**`flush: 'post'`**：預設 watch 在 DOM 更新前執行，加上 `flush: 'post'` 表示等 DOM 更新完再執行：

```ts
watch(wrapperStyle, async () => {
  await nextTick()  // 等待 DOM 更新
  // 這裡可以讀到最新的 DOM
}, { deep: true, flush: 'post' })
```

---

## toRefs：把 reactive 物件拆成多個 ref

```ts
import { reactive, toRefs } from 'vue'

const props = reactive({ img: '', color: '#fff' })
const { img, color } = toRefs(props)

// 現在 img 和 color 各是一個 ref
// 修改 img.value 等同修改 props.img
```

本專案用 `toRefs(props)` 把 Props 拆開，方便在 `watch` 裡監聽個別 prop。

---

## nextTick：等待 DOM 更新完

Vue 更新 DOM 是非同步的（批次更新）。如果你改了資料之後馬上讀 DOM，會讀到舊的。

```ts
import { nextTick, ref } from 'vue'

const show = ref(false)

async function showAndMeasure() {
  show.value = true
  // 此時 DOM 還沒更新
  await nextTick()
  // 此時 DOM 已更新，可以安全讀取元素尺寸
}
```

---

## defineProps：宣告元件的輸入

```ts
// 使用 TypeScript interface 定義 Props 型別
interface Props {
  img?: string      // ? 表示可選（不傳也沒關係）
  color?: string
  width: number     // 沒有 ? 表示必填
}

const props = defineProps<Props>()
```

**`withDefaults`**：同時設定預設值：

```ts
const props = withDefaults(defineProps<Props>(), {
  img: '',
  color: '#fff',
})
```

> **TypeScript 穿插說明 — interface**  
> `interface` 用來描述一個物件的「形狀」（有哪些屬性、各是什麼型別）。  
> `?` 表示這個屬性是可選的，沒傳時值是 `undefined`。

---

## defineEmits：宣告元件可發出的事件

```ts
const emit = defineEmits<{
  (e: 'img-load', data: { type: string; message: string }): void
  (e: 'real-time', payload: object): void
}>()

// 使用
emit('img-load', { type: 'success', message: '載入成功' })
```

父元件監聽：

```html
<VueCropper @img-load="handleLoad" />
```

---

## defineExpose：讓父元件可以呼叫子元件的方法

`<script setup>` 的內容預設對外不可見。用 `defineExpose` 明確「開放」：

```ts
defineExpose({
  getCropData,   // 父元件可以呼叫 cropperRef.value.getCropData()
  rotateLeft,
  rotateRight,
})
```

---

## onMounted / onUnmounted：生命週期

```ts
import { onMounted, onUnmounted } from 'vue'

onMounted(() => {
  // 元件插入 DOM 後執行
  // 可以在這裡讀取 DOM 元素尺寸、綁定事件
})

onUnmounted(() => {
  // 元件從 DOM 移除時執行
  // 用來清理：移除事件監聽、取消計時器、釋放資源
})
```

---

## ref 取得 DOM 元素

在 template 加上 `ref="名稱"`，在 JS 裡就能取到這個 DOM 元素：

```vue
<script setup>
import { ref, onMounted } from 'vue'
const boxRef = ref<HTMLElement>()

onMounted(() => {
  console.log(boxRef.value)  // 這裡才有值（DOM 已插入）
})
</script>

<template>
  <div ref="boxRef"></div>
</template>
```

本專案用這個方式取得裁剪容器、圖片元素、裁切框的 DOM 節點，來綁定拖曳事件。

---

## useSlots：讀取插槽內容

```ts
import { useSlots } from 'vue'

const slots = useSlots()

// 判斷使用者有沒有傳入 loading 插槽
if (slots.loading) {
  // 使用自訂 loading
} else {
  // 使用預設 loading
}
```

---

## template 語法速覽

| 語法 | 說明 | 範例 |
|------|------|------|
| `{{ }}` | 插值（顯示變數）| `{{ name }}` |
| `v-if` | 條件渲染 | `v-if="show"` |
| `v-show` | 顯示/隱藏（保留 DOM）| `v-show="show"` |
| `:attr` | 動態綁定屬性 | `:style="styleObj"` |
| `@event` | 事件監聽 | `@click="handleClick"` |
| `ref` | 取得 DOM 節點 | `ref="boxRef"` |
| `<slot>` | 插槽（讓使用者自訂內容）| `<slot name="loading">` |

---

## 小結

學完這篇，你知道了：

- `ref` / `reactive`：讓資料驅動畫面
- `computed`：自動計算的衍生值
- `watch`：監聽資料變化
- `defineProps` / `defineEmits`：元件的輸入輸出
- `defineExpose`：開放方法給父元件
- `onMounted` / `onUnmounted`：生命週期
- `ref` 取 DOM：讀取真實元素

TypeScript 的部分（`interface`、`type`、泛型等）會在後續各篇遇到時再說明。

下一篇：[03 — lib/ 目錄規劃與設計思路](./03-lib-structure.md)
