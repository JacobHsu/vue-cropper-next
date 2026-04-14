# 22 — src/pages/ + md.config.ts（Markdown 渲染）

## 這套機制做什麼？

展示網站的大部分頁面是 Markdown 檔案（`.md`）。

這些 Markdown 可以直接包含 Vue 元件語法（`<VueCropper />`、`<LangBlock />`），甚至內嵌可以實際運行的互動範例。

這是透過 `unplugin-vue-markdown`（Vite 外掛）實現的：**把 Markdown 編譯成 Vue 元件**。

---

## 技術架構

```
src/pages/DemoBasic.md
       ↓  unplugin-vue-markdown
Vue SFC（單檔元件）
       ↓  @vitejs/plugin-vue
瀏覽器可執行的 JS
```

轉換後，`.md` 檔案變成一個可以被 Vue Router `import` 的 Vue 元件：

```ts
// src/main.ts
{ name: 'DemoBasic', path: '/demo-basic', component: () => import('./pages/DemoBasic.md') }
```

---

## md.config.ts：自訂 Markdown 渲染

```ts
import container from 'markdown-it-container'
import MarkdownIt from 'markdown-it'
import striptags from './strip-tags.js'

const markdownRenderer = new MarkdownIt()

const convertHtml = (str: string) => {
  return str.replace(/(&#x)(\w{4});/gi, $0 =>
    String.fromCharCode(parseInt(encodeURIComponent($0).replace(/(%26%23x)(\w{4})(%3B)/g, '$2'), 16))
  )
}

export default (md: MarkdownIt) => {
  md.use(container, 'demo', {
    validate: (params: string) => params.trim().match(/^demo\s*(.*)$/),
    render: (tokens, idx) => {
      if (tokens[idx].nesting === 1) {
        const html = convertHtml(striptags(tokens[idx + 1].content, 'script'))
        return `<demo>
                  <template #demo>${html}</template>
                  <template #sourceCode>`
      }
      return '</template></demo>'
    }
  })
}
```

### `markdown-it-container`

`markdown-it-container` 是 `markdown-it` 的外掛，讓 Markdown 支援自訂容器語法：

```md
::: demo
內容
:::
```

這個外掛處理 `:::` 語法，把裡面的內容傳給我們的 `render` 函數。

### `striptags`

```ts
const html = convertHtml(striptags(tokens[idx + 1].content, 'script'))
```

`striptags(content, 'script')` 取出 `<script>` 標籤裡的內容（移除其他 HTML 標籤）。

**為什麼要這樣處理？**

`::: demo` 裡的內容同時包含：
1. Template 程式碼（要顯示在 `#demo` 插槽，實際運行）
2. Script 程式碼（也要顯示在 `#sourceCode` 插槽，但作為原始碼展示）

`striptags` 把 script 部分提取出來，讓兩個插槽都有需要的內容。

### `convertHtml`

Markdown 渲染時，HTML 實體（如 `&#x7B;` = `{`）會被轉義。這個函數把轉義後的實體重新還原成原始字符，讓展示的原始碼看起來正確。

---

## 完整轉換範例

### 原始 Markdown

```md
:::demo
```html
<vue-cropper ref="cropper" :img="img" />
<demo-image-switch v-model="img" />
```

```js
<script setup>
import { ref } from 'vue'
const cropper = ref()
const img = ref('')
</script>
```
:::
```

### 轉換後的 HTML

```html
<demo>
  <template #demo>
    <vue-cropper ref="cropper" :img="img" />
    <demo-image-switch v-model="img" />
  </template>
  <template #sourceCode>
    <pre class="hljs"><code>
      <!-- highlight.js 處理後的原始碼 HTML -->
    </code></pre>
  </template>
</demo>
```

`Demo.vue` 接收這兩個插槽，顯示互動範例 + 可展開/收起的原始碼。

---

## vite.config.ts 裡的 Markdown 設定

```ts
Markdown({
  markdownItOptions: {
    html: true,       // 允許 Markdown 裡的 HTML 標籤
    linkify: true,    // 自動把 URL 文字變成連結
    typographer: true, // 排版美化（引號等）
    highlight: (str: string, lang: string) => {
      if (lang && hljs.getLanguage(lang)) {
        try {
          return '<pre class="hljs"><code>' +
            hljs.highlight(str, { language: lang }).value +
            '</code></pre>'
        } catch (__) {}
      }
      return '<pre class="hljs"><code>' + markdownRenderer.utils.escapeHtml(str) + '</code></pre>'
    }
  },
  markdownItSetup(md) {
    mdConfig(md)  // 應用 md.config.ts 的設定（::: demo 語法）
  },
  wrapperClasses: 'markdown-body',  // 整個 Markdown 內容包在這個 class 裡
}),
```

**`highlight.js`**：程式碼語法高亮，支援 100+ 種程式語言，自動為 keywords、字串、註解等上色。

**`wrapperClasses: 'markdown-body'`**：所有 Markdown 頁面的內容都在 `.markdown-body` 這個 div 裡，可以統一設定樣式（字體、行距、標題大小等）。

---

## Markdown 頁面的結構

```
src/pages/
├── Home.vue              ← 首頁（.vue 不是 .md，因為有複雜的互動邏輯）
├── Guide.md              ← 快速開始
├── Props.md              ← Props 說明（表格）
├── Methods.md            ← Methods 說明
├── Event.md              ← Events 說明
├── Changelog.md          ← 更新日誌
├── DemoBasic.md          ← 基礎範例
├── DemoLoading.md        ← 自訂 Loading 範例
├── DemoDrag.md           ← 拖曳上傳範例
├── DemoCrop.md           ← 裁切框操作範例
├── DemoImg.md            ← 圖片模式範例
├── DemoFilter.md         ← 濾鏡範例
├── DemoRotate.md         ← 旋轉範例
├── DemoRealtime.md       ← 即時預覽範例
├── DemoExport.md         ← 導出選項範例
└── DemoAll.md            ← 完整功能展示
```

---

## 一個完整的 Markdown 頁面（DemoBasic.md 節選）

```md
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

...

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
<crop-export-panel :cropper="cropper" :display-width="220" :display-height="220" />
```

```js
<script setup>
  import { ref } from 'vue'
  const cropper = ref()
  const img = ref('')
</script>
```
:::
```

### 這個頁面渲染後：

1. `<LangBlock lang="zh">` 裡的中文內容，語言切換到英文時隱藏
2. `<LangBlock lang="en">` 裡的英文內容，語言切換到中文時隱藏
3. `:::demo` 塊 → 轉換成 `<Demo>` 元件 → 顯示可互動的裁剪元件 + 原始碼

---

## 小結

| 技術 | 用途 |
|------|------|
| `unplugin-vue-markdown` | 把 `.md` 編譯成 Vue SFC |
| `markdown-it` | Markdown 解析核心 |
| `markdown-it-container` | 支援 `:::demo` 容器語法 |
| `highlight.js` | 原始碼語法高亮 |
| `md.config.ts` | 把 `:::demo` 轉換成 `<Demo>` 插槽語法 |

這套機制讓文件維護非常方便：只需要寫 Markdown，加上 `:::demo` 就有可互動的範例，不需要手動寫繁瑣的 Vue 元件。

下一篇：[23 — vite.config.ts（展示網站建置設定）](./23-vite-docs-config.md)
