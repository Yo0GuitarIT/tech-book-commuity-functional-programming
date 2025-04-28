---
marp: true
theme: gaia
class:
    - invert
paginate: true
---

# <!-- fit --> 《簡約的軟體開發思維：用 Functional Programming 重構程式》<br> -> 以 Javascript 為例

Ch6 在變動的程式中讓資料保持不變
Ch7 讓不變性不受外來程式破壞

Speaker： Yo0
Note Taker：Lois
2025/05/01 @Tech-Book-Community

---

# <!-- fit --> Ch6 在變動的程式中讓資料保持不變

-   『寫入時複製(copy-on-write)』來確保資料不被改變。
-   對陣列及物件實作寫入時複製。
-   確保寫入時複製能在深度巢狀資料上生效

---

## <!-- fit --> 6.1 在任何操作中的資料都能具有不變性嗎？

> 程式會先將購物車陣列複製一份，在對副本進行修改並傳回。
> 想想看，你能讓之前相關的資料保持不變嗎？

| 購物車操作                                                                                                                                                                                   | 商品物件的操作                                |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| 1. 取得商品數量<br>2. 取得商品名稱<br>3. 加入商品 (已實作完成) <br>4. 根據名稱將指定商品刪除<br>5. `根據名稱更新指定商品的數量`<br>`(作用於巢狀資料的操作)`<br>6. 根據名稱更新指定商品的價格 | 1. 設定價格<br>2. 取得價格<br>3. 取得商品名稱 |

---

## 巢狀資料結構(nested)

-   某個資料結構中包含另一資料結構時，成為巢狀資料結構。

```javascript
const arr = [1, [2, 3], 4];

const obj = {
    name: "Yo0",
    band: {
        name: "Linkin Park",
        genre: "Rock",
    },
};
```

---

-   若上述巢狀資料結構持續很多層，稱為**深度巢狀（deeply nested）**

```javascript
const deeplyNested = {
    a: {
        b: {
            c: {
                d: {
                    e: "Hello",
                },
            },
        },
    },
};
```

---

## 6.2 將操作分為『讀取』、『寫入』與讀取兼寫入

#### 一項操作可能是『讀取』或『寫入』

| 讀取                                   | 寫入                                                                     |
| -------------------------------------- | ------------------------------------------------------------------------ |
| - 讀取資料中的訊息 <br> - 不會改變資料 | - 會改變資料                                                             |
| 容易實踐不變性                         | `必須使用特殊技巧在寫入上實作不變性，防止相同資料的函式受數值變化影響。` |

-> 當函式只透過引數讀取資料，該函式屬於 Calculation。

---

### 購物車操作

| 讀取                                 | 寫入                                                                                                             |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| 1. 取得商品數量 <br> 2. 取得商品名稱 | 3. 加入商品 <br> 4. 根據名稱將指定商品刪除 <br> 5. 根據名稱更新指定商品的數量 <br> 6. 根據名稱更新指定商品的價格 |

-   以購物車操作為例，想要實作不變性
    在寫入時選擇的方式是`寫入時複製(copy-on-write)`

-   🙋 能不能讓操作既『讀取』又『寫入』？ 🆗

---

商品物件的操作

| 寫入        | 讀取                           |
| ----------- | ------------------------------ |
| 1. 設定價格 | 2. 取得價格<br>3. 取得商品名稱 |

預設資料是不可變的 FP 語言

```
- Haskell           # 其他語言可能有不變資料結構，但預設使用可變資料。
- Clojure           # 來有一些 FP 語言完全仰賴使用者自行實作不變性。
- Elm
- Purescript
- Erlang
- Elixir
```

---

## 6.3 實作『寫入時複製』的三步驟

1️⃣ 產生副本
2️⃣ 修改副本（修改幾次都沒問題～）
3️⃣ 傳回副本

```javaScript
function add_element_last(array, elem){     // array -> 是我們想改變的陣列
    var new_array =array.slice();           // 1. 產生副本
    new_array.push(elem);                   // 2. 修改副本
    return new_array;                       // 3.傳回副本
}
```

---

### 💡 避免元陣列改變的原因

1. 複製不會改變原始資料。
2. 副本是函式內的區域變數，只有該函式能修改，外部程式碼無法存取。
3. 改完的副本會透過 `return` 離開函式，其內容不會再有變化。

---

### <!--fit-->`add_element_last` 到底是『讀取』還是『寫入』操作呢？

```javaScript
function add_element_last(array, elem){     // array -> 是我們想改變的陣列
    var new_array =array.slice();           // 1. 產生副本
    new_array.push(elem);                   // 2. 修改副本
    return new_array;                       // 3.傳回副本
}
```

Ans: `讀取`

-   該函式不修改既有資料。
-   只會將複製副本資訊傳回。（不修改既有資料）

✏️ **一項`寫入`操作，能藉由<複製>變成`讀取`**

---

## 6.4 利用『寫入時複製』將『寫入』變成『讀取』

修改購物車的操作：

```javaScript
function remove_item_by_name(cart, name) {
    var idx = null;
    for(var i = 0; i < cart.length; i++) {
        if(cart[i].name === name)
        idx = i;
    }
    if(idx !== null)
        cart.splice(idx, 1); // 在索引 idx 上，移除一項商品。 ❌會修改購物車陣列
}

// NOTE:
// 由於有 cart.splice()，remove_item_by_name()就會修改原是資料（寫入）。
// 將全域變數 shopping_cart 傳入該函式就會改變本質。
```

---

```javaScript
function remove_item_by_name(cart, name) {                                  //  1. 產生副本
    var new_cart = cart.slice();    // 複製 cart 引數，並存入一個區域變數 new_cart 中。
    var idx = null;
    for(var i = 0; i < cart.length; i++) {
        if(cart[i].name === name)
            idx = i;
    }
    if(idx !== null)
        cart.splice(idx, 1);
    }
```

```javaScript
function remove_item_by_name(cart, name) {                           // 2. 對副本進行修改
    var new_cart = cart.slice();
    var idx = null;
        for(var i = 0; i < new_cart.length; i++) {  // ✅ new_cart ❌ cart(原陣列)
            if(new_cart[i].name === name)
                idx = i;
        }
  if(idx !== null)
    new_cart.splice(idx, 1);
}
```
---

```javaScript
function remove_item_by_name(cart, name) {                                  // 3. 傳回副本
    var new_cart = cart.slice();
    var idx = null;
    for(var i = 0; i < new_cart.length; i++) {
        if(new_cart[i].name === name)
        idx = i;
    }
    if(idx !== null)
        new_cart.splice(idx, 1);
    return new_cart; // ⬅️ 傳回副本
}
```

- `remove_item_by_name` 寫入時複製的實作已完成
- 該函式由`寫入`變成`讀取`。
- 接下來是修改呼叫該函式的手段。

---

```javaScript
// 原本程式碼
function delete_handler(name) {  
    remove_item_by_name(shopping_cart, name);   // 此函式會更改全域變數
    var total = calc_total(shopping_cart);
    set_cart_total_dom(total);
    update_shipping_icons(shopping_cart);
    update_tax_dom(total);
}

⬇️ ⬇️ ⬇️
// 加入寫入時複製
function delete_handler(name) {
    // 現在我們將 remove_item_by_name()的傳回值另外指定給全域變數
    shopping_cart = remove_item_by_name(shopping_cart, name); 
    var total = calc_total(shopping_cart);
    set_cart_total_dom(total);
    update_shipping_icons(shopping_cart);
    update_tax_dom(total);
}
// `remove_item_by_name()` 可能不只 `delete_handler()` 使用可以將原本系統中類似的功能做相同的重構。
```



---

## 6.5 對比實作『寫入時複製』前後的程式碼

---

## 6.6 將實作『寫入時複製』的操作普適化

---

## 6.7 簡介 JavaScript 陣列

---

## 6.8 如果操作既是『讀取』也是『寫入』怎麼辦？

---

## 6.9 拆解同時『讀取』與『寫入』的函式

---

## 6.10 讓一個函式傳回兩個值

---

## 6.11 讀取不可變資料結構屬於 Calculations

---

## 6.12 程式中包含隨時間而變的狀態

---

## 6.13 不可變資料的效率已經夠高

---

## 6.14 作用在物件上的寫入時複製操作

---

## 6.15 簡介 JavaScript 物件

---

## 6.16 將巢狀資料的『寫入』轉換成『讀取』

---

## 6.17 巢狀資料中的哪些都星需要複製？

---

## 6.18 將『淺拷貝』與『結構共享』視覺化

---

# Ch7 讓不變性不受外來程式破壞

---

## 7.1 是用既有程式(legacy code)時的不變性

---

## 7.2 寫入時複製函式需與未實作不變性的函式互動

---

## 7.3 防禦型複製能守護資料不變性

---

## 7.4 實作防禦型複製

---

## 7.5 防禦型複製的原則

---

## 7.6 將不受信任的程式包裝起來

---

## 7.7 你或許看過的防禦型複製

---

## 7.8 比較『寫入時複製』與『防禦型複製』

---

## 7.9 深拷貝所需資源較淺拷貝高

---

## 7.10 以 JavaScript 實作深拷貝很困難

---

## ７.11 想像『寫入時複製』與『防禦型複製』之間的對話

---

## Thank You
