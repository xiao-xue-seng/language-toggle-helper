# language-toggle-helper

## 祈請大寶恩師加持

此【中藏(多語)切換】模組是為了與其它「使用月光藏 API 的平台」共用而獨立出來，在設計上會考慮"中立性"，並不是特別針對 Vue 而設計。開發者不一定需要使用此模組，也可依據下文資訊，自行切換語言段落，提升與平台的耦合性，惟無法同步更新。

此模組僅負責資料及邏輯運算，UI 切換開關由各平台（如 HTML、Vue、React 等）自行實作。

### 🟩 功能版號：

「中藏(多語)切換」功能版本訊息以 `<meta>` 標籤標記：

```html
<meta name="lang-toggle-version" content="1.0" />
```

### 🟩 「可切換段落」屬性：

API 返回的 HTML 內容中，可切換段落會帶有 `data-toggle-lang` 屬性，值為該段落的語言代號。例如：

```html
<p class="scripture-modern" data-toggle-lang="modern">如果因此不害怕惡趣也不皈依的話...。</p>
```

### 🟩 支援語言：

目前支援以下語言：

| 代號        | 語言 |
| ----------- | ---- |
| `bo`        | 藏文 |
| `zh`        | 中文 |
| `classical` | 古文 |
| `modern`    | 白話 |

模組的 `dataList` 屬性提供所有語言的狀態資訊，每個語言項目包含：

| key         | value     | 意義                 | 讀寫 | 用途                 |
| ----------- | --------- | -------------------- | ---- | -------------------- |
| `lang`      | `String`  | 語言代號             | 唯讀 | 識別                 |
| `enabled`   | `Boolean` | 開關狀態             | 讀寫 | 綁定 UI 切換開關的值 |
| `available` | `Boolean` | 是否有此語言可供切換 | 讀寫 | (不應直接修改)       |
| `label`     | `String`  | 文字標籤             | 唯讀 | UI 開關的標籤        |

### 🟩 使用步驟：

概述步驟後，再詳述：

1. 創建模組實例，可自訂開關預設值。
2. 綁定本文元素，使模組可控制語言段落顯示。
3. 依據 `availableLanguages` 清單建立 UI 切換開關。
4. 將開關狀態綁定至 `availableLanguages` 內 `enabled` 屬性。
5. 本文內容變更時：
   - 呼叫 `reset()` 重置狀態。
   - 若本文元素更換，則重新綁定新元素。
   - 呼叫 `updateAvailabilityByHtml()` 或 `updateAvailabilityByElement()` 更新可用語言狀態。
   - 呼叫 `applyLanguageToggles()` 以同步開關狀態。
6. 訂閱變更通知以監測語言狀態。
7. 銷毀實例。

#### 🟢 創建模組實例：

```javascript
let languageToggleHelper = LanguageToggleHelper();
```

可自訂「開關預設值」：

```javascript
let languageToggleHelper = LanguageToggleHelper(new SwitchConfig({ bo: false }));
```

#### 🟢 綁定本文元素：

要讓模組控制的根元素稱為「本文元素」，UI 開關變化時，模組會控制該元素內的語言段落顯示。

在挑選本文元素時，越貼近實際文字區塊，執行時效率越好。譬如假使將整個`<body>`當作本文元素，雖然也是可以運作，但每次切換時，都必須在許多不相關的元素中搜尋，效率自然不高。

可使用 `contextElement` 屬性或 `bindContextElement()` 進行綁定，兩者等效：

```javascript
languageToggleHelper.contextElement = yourContextElement;
languageToggleHelper.bindContextElement(yourContextElement);
```

解除綁定，可給予任何「非 HTMLElement」的值：

```javascript
languageToggleHelper.contextElement = false;
languageToggleHelper.contextElement = null;
```

`bindContextElement()` 返回自身，方便串接：

```javascript
languageToggleHelper.bindContextElement(null).reset();
languageToggleHelper.bindContextElement(yourContextElement).applyLanguageToggles();
```

必須確保本文元素已渲染完成後再進行綁定。例如在 Vue `mounted()` 的 `$nextTick()` 內執行：

```javascript
  mounted() {
    this.$nextTick(() => {
      languageToggleHelper.bindContextElement(this.$refs.context);
    });
  },
```

如果所提供的參數不是 `HTMLElement`，則 `contextElement = false`。

#### 🟢 依據 availableLanguages 建立 UI 切換開關：

`availableLanguages` 屬性提供當前本文內可供切換的語言清單，它其實是總表 `dataList` 的篩選結果，對其中一者的內容作修改，等同於對另一者修改。

利用 `availableLanguages` 決定 UI 需要渲染哪些開關。以 Vue 為例：

{% raw %}

```html
<template>
  <div class="lang-switch-group round-btn">
    <div v-for="item in availableLanguages" :key="item.lang" class="lang-switch">
      <label>
        {{item.label}}
        <el-switch v-model="item.enabled" inactive-color="darkgray" active-color="gold" />
      </label>
    </div>
  </div>
</template>
```

{% endraw %}

上述範例模板中已將 切換開關的值 雙向綁定到 `availableLanguages` 列表的 `enabled`。
由於此模組的 `availableLanguages` 非 Vue 響應變數，所以必需自行定義一個響應變數，並訂閱 `onAvailableLanguagesChange()` 更新狀態：

定義響應變數：

```javascript
  data() {
    return {
      availableLanguages: Vue.observable([])
    };
  },
```

訂閱變更通知。將模組的 `availableLanguages` 內容填入響應變數中：

```javascript
  created() {
    // 初始化數據，清空並填入新值。availableLanguages是響應變數，所以不能直接賦予新物件。
    this.availableLanguages.splice(
      0,
      this.availableLanguages.length,
      ...languageToggleHelper.availableLanguages
    );

    // 登記 callback，當 availableLanguages 發生變化時更新響應式數據
    this.languageChangeCallback = updatedLanguages => {
      this.availableLanguages.splice(0, this.availableLanguages.length, ...updatedLanguages);
    };
    languageToggleHelper.onAvailableLanguagesChange(this.languageChangeCallback);
  },
```

移除訂閱方式：

```javascript
offAvailableLanguagesChange(callback);
```

#### 🟢 開關狀態記憶：

模組會將使用者的語言開關狀態儲存於 `localStorage` (索引名稱為 `AMEC_LanguageToggleState`)，以便跨文章維持相同偏好設定。

#### 🟢 本文內容變更：

當本文內容變更時，需執行以下操作：

1. 呼叫 `reset()` 重置開關的狀態及可用性。
2. 若本文元素更換，應重新綁定新元素。
3. 更新語言的可用性，有兩種方式可選：
   1. 本文內容 HTML 取得後，呼叫 `updateAvailabilityByHtml()`。此方式不需要內容已渲染完成，適用於希望盡早得知語言可用性的場合。
   2. 本文內容初次渲染完成後，呼叫 `updateAvailabilityByElement()`。它會依據所綁定的本文元素來更新。
4. 在本文內容初次渲染完成後，呼叫 `applyLanguageToggles()` 以同步開關狀態。

以 Vue 為例：

```javascript
  watch: {
    "selectedVolumeData.context": {
      handler(newValue) {
        // 假設綁定的本文元素每次都會被重建，則可以先斷開。
        // 每次文章內容變更都重置切換開關，避免使用者誤會、困擾。
        // 先斷開綁定，再重置，可以加速內部作業。
        languageToggleHelper.bindContextElement(null).reset();

        if (!newValue) return;

        // 如果使用HTML版更新「可切換語言清單」：
        // languageToggleHelper.updateAvailabilityByHtml(newValue);

        // 以下是需要等元素更新後才處理的部分。因為要查詢DOM。
        this.$nextTick(() => {
          languageToggleHelper
            // 綁定新的本文元素
            .bindContextElement(this.$refs.context)
            // 如果用元素版更新「可切換語言清單」：
            .updateAvailabilityByElement()
            // 套用首次開關狀態
            .applyLanguageToggles();

          // 其他需要displayContext變化後，執行副作用的部分
          // ....
        });
      },
      immediate: true
    }
  },
```

#### 🟢 訂閱語言變更通知：

此模組提供「可切換語言變更」通知，你可以登記處理函數，在可用語言變更時進行適當處理，例如在 Vue 中更新響應變數。

登記：

```javascript
onAvailableLanguagesChange(callback);
```

通知時，`callback()` 會傳入一個參數：更新後的 `availableLanguages` 陣列。

移除登記：

```javascript
offAvailableLanguagesChange(callback);
```

#### 🟢 銷毀模組：

最後，模組提供一個銷毀方法，可徹底清除引用，防止殘留問題。以 Vue 為例：

```javascript
  // 這是Vue2的，Vue3已改為beforeUnmount
  beforeDestroy() {
    languageToggleHelper.destroy();
  }
```

## 🟩 其他技術資訊：

以下資訊供「自行實現切換功能」時參考。

🟢 `data-toggle-lang="x"` 表示該段落不參與語言切換。這是用於編輯時期，搭配自斷判斷語言功能的一種標籤。所以，你並不需要創建一個「x」語言切換開關。

🟢 自動模式 `<meta>` 標籤僅於編輯時使用，無前端作用：

```html
<meta name="lang-toggle-mode" content="auto" />
```
