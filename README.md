# language-toggle-helper

## 祈請大寶恩師加持

此【中藏(多語)切換】模組是為了與其它 使用月光藏 API 的平台 共用而獨立出來，在設計上會考慮"中立性"，不是特別針對 Vue 設計。開發者不一定需要使用此模組，可依據下文所提供的資訊，自行切換各語言段落，這樣在耦合性上會更好。只不過差別在於，會沒有同步更新。

此模組只負責資料及邏輯運算，UI 切換開關 的創建由各平台（例如 傳統 HTML、Vue、React 等）自行實作。

### 🟩 可切換段落的屬性：

從 API 所取得的 HTML 中，「可切換的段落」會被賦予 `data-toggle-lang` 屬性，它的值就是語言代號。例如：

```html
<p class="scripture-modern" data-toggle-lang="modern">如果因此不害怕惡趣也不皈依的話...。</p>
```

### 🟩 語言種類：

目前支援的語言種類有：

| 代號        | 語言 |
| ----------- | ---- |
| `bo`        | 藏文 |
| `zh`        | 中文 |
| `classical` | 古文 |
| `modern`    | 白話 |

從模組的 `dataList` 屬性，可得知所有"語言"的資料及當前狀態。它所傳回的陣列中，每個項目是一種"語言"，而每個語言項目中包含以下資訊：

| key         | value     | 意義                   | 讀寫 | 用途                 |
| ----------- | --------- | ---------------------- | ---- | -------------------- |
| `lang`      | `String`  | 語言代號               | 唯讀 | 識別                 |
| `enabled`   | `Boolean` | 開關狀態               | 讀寫 | 綁定 UI 切換關關的值 |
| `available` | `Boolean` | 是否有這個語言可供切換 | 讀寫 | (不應該直接改變它)   |
| `label`     | `String`  | 文字標籤               | 唯讀 | UI 開關的文字標籤    |

### 🟩 使用步驟：

先列出所有步驟，再逐一說明。

- 創建實例，可自訂開關預設值。
- 綁定 本文元素，讓模組可以控制其中的語言段落。
- 依據 `availableLanguages` 列表 建立 UI 切換開關。
- 將 切換開關的值 綁定到 `availableLanguages` 列表項目 的 `enabled`。
- 當本文內容變更時：
  - 呼叫 `reset()`。
  - 如果更新本文時會造成本文元素更換，應重新綁定新元素。
  - 當本文內容 HTML 取得時，呼叫 `updateAvailabilityByHtml()`。(二選一)
  - 當本文內容渲染完成時，呼叫 `updateAvailabilityByElement()`。(二選一)
  - 當本文內容渲染完成時，呼叫 `applyLanguageToggles()` 以套用首次開關狀態。
- 依需要訂閱變更通知。

#### 🟢 創建實例

```javascript
let languageToggleHelper = LanguageToggleHelper();
```

自訂「開關預設值」的範例

```javascript
let languageToggleHelper = LanguageToggleHelper(new SwitchConfig({ bo: false }));
```

#### 🟢 綁定本文元素

「本文元素」是要讓模組控制的根元素，當 UI 開關變化時，模組會控制此元素中段落的顯示狀態。本文元素越貼近實際文字區塊，執行時效率越好。譬如假使將`<body>`當作本文元素，雖然也是可以運作，但每次切換開關都必須在許多不相關的元素中搜尋。

綁定的方式可對 `contextElement` 屬性賦值，或使用 `bindContextElement()`，兩者等效。

```javascript
languageToggleHelper.contextElement = yourContextElement;
languageToggleHelper.bindContextElement(yourContextElement);
```

若要解除綁定，可給予任何「非 HTMLElement」的值。

```javascript
languageToggleHelper.contextElement = false;
languageToggleHelper.contextElement = null;
```

`bindContextElement()`會傳回模組物件自身，方便串接。例如：

```javascript
languageToggleHelper.bindContextElement(null).reset();
languageToggleHelper.bindContextElement(yourContextElement).applyLanguageToggles();
```

必須確保本文元素已經渲染完成後，才能進行綁定。以 Vue 為例，可以在 `mounted()` 的 `$nextTick()` 內進行：

```javascript
  mounted() {
    this.$nextTick(() => {
      languageToggleHelper.bindContextElement(this.$refs.context);
    });
  },
```

如果所提供的參數不是 `HTMLElement`，則 `contextElement = false`。

#### 🟢 依據 availableLanguages 建立 UI 切換開關

透過 `availableLanguages` 屬性，可取得「當前本文中可供切換的語言清單」，它其實是總表 `dataList` 的篩選結果，對其中一者的內容作修改，等同於對另一者修改。

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

上述範例模板中已將 切換開關的值 雙向綁定到 availableLanguages 列表的 enabled。

此模組的`availableLanguages`屬性本身並不是 Vue 響應變數，所以需要自己定義一個，然後再透過訂閱模組的「變更通知 `onAvailableLanguagesChange()`」來更新。

定義一個響應變數：

```javascript
  data() {
    return {
      availableLanguages: Vue.observable([])
    };
  },
```

訂閱變更通知以更新響應變數：

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

如果有需要，可以使用 `offAvailableLanguagesChange(callback)` 來移除訂閱。

#### 🟢 開關狀態記憶

模組會自動將使用者操作的開關狀態紀錄在 `localStorage` 中，索引名稱為 `AMEC_LanguageToggleState`。
在創建模組時，會優先採用「使用者狀態值」作為開關初始值，然後才是「自訂預設值」(如果有的話)。
「使用者狀態值」會跨文章保持，假使當前關閉了藏文，切換到下一篇文章時仍會是關閉藏文。

#### 🟢 本文內容變更

當本文內容變更時，需要做以下幾件事：

1. 呼叫 `reset()` 重置開關的狀態及可用性。
2. 如果你的應用在變更內容時會更換本文元素，則應在新元素完成渲染後，重新綁定新元素。
3. 更新語言的可用性，有兩種方式，可選其中之一：
   1. 本文內容 HTML 取得後，呼叫 `updateAvailabilityByHtml()`。這可以在內容實際渲染之前進行。如果你希望盡早得知有哪些語言可切換，可以用這個方法。
   2. 本文內容初次渲染完成後，呼叫 `updateAvailabilityByElement()`。它會依據綁定的本文元素來更新。
4. 在本文內容初次渲染完成後，必須呼叫 `applyLanguageToggles()` 以套用第一次的開關值，這樣才能讓本文的顯示狀態與開關同步。

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

#### 🟢 訂閱變更通知

此模組提供「可切換語言變更」通知，你可以訂閱處理函數，在可用語言變更時進行適當處理，例如在 Vue 中更新響應變數。

登記：

```javascript
onAvailableLanguagesChange(callback);
```

通知時，`callback()` 會傳入一個參數：更新後的 `availableLanguages` 陣列。

移除登記：

```javascript
offAvailableLanguagesChange(callback);
```

最後，模組提供一個銷毀方法，可徹底清除引用，防止殘留問題。以 Vue 為例：

```javascript
  // 這是Vue2的，Vue3已改為beforeUnmount
  beforeDestroy() {
    languageToggleHelper.destroy();
  }
```
