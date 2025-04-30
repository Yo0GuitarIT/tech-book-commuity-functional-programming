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
function remove_item_by_name(cart, name) {                                      //  1. 產生副本
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
function remove_item_by_name(cart, name) {                                  // 2. 對副本進行修改
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

-   `remove_item_by_name` 寫入時複製的實作已完成
-   該函式由`寫入`變成`讀取`。
-   接下來是修改呼叫該函式的手段。

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

6.5 對比實作『寫入時複製』前後的程式碼 - 原始程式碼

```javaScript
function remove_item_by_name(cart, name) {  // 原始程式碼
    var idx = null;
    for(var i = 0; i < cart.length; i++) {
        if(cart[i].name === name)
        idx = i;
    }
    if(idx !== null)
        cart.splice(idx, 1);
}

function delete_handler(name) {
    remove_item_by_name(shopping_cart, name);
    var total = calc_total(shopping_cart);
    set_cart_total_dom(total);
    update_shipping_icons(shopping_cart);
    update_tax_dom(total);
}
```

---

6.5 對比實作『寫入時複製』前後的程式碼 - 寫入時複製的版本

```javascript
function remove_item_by_name(cart, name) {
    var new_cart = cart.slice(); 🆕
    var idx = null;
    for(var i = 0; i < new_cart.length; i++) { 🆕
        if(new_cart[i].name === name) 🆕
        idx = i;
    }
    if(idx !== null)
        new_cart.splice(idx, 1); 🆕
    return new_cart;    🆕
}

function delete_handler(name) {
    shopping_cart = remove_item_by_name(shopping_cart, name); 🆕
    var total = calc_total(shopping_cart);
    set_cart_total_dom(total);
    update_shipping_icons(shopping_cart);
    update_tax_dom(total);
}
```

---

## 6.6 將實作『寫入時複製』的操作普適化

`寫入時複製` 的操作可能被用在多個函式中。
為了增加可重複使用性，可以將相關程式碼擷取出來。

```javaScript
// 原始程式碼
function removeItem(array,idx,count){
    array.splice(idx,count);
}

// 寫入時複製
function removeItem(array, idx, count){
    var copy = array.slice(); ⬅️
    copy.splice(idx, count); ⬅️
    return copy; ⬅️
}
```

---

```javaScript
function remove_item_by_name(cart, name){           // 之前的實作
    var new_cart = cart.slice(); //由於 removeItems() 會複製陣列，所以不需要此行
    var idx = null;
    for( var i = 0; i < new_cart.length; i++){
        if(new_cart[i] === name)
            idx = i;
    }
    if(idx !== null)
        new_cart.splice(idx,i);
    return new_cart;
}


function remove_item_by_name(cart, name){           // 使用 removeItems()
    var idx = null;
    for( var i = 0; i < cart.length; i++){
        if(cart[i] === name)
            idx = i;
    }
    if(idx !== null)
        return removeItems(cart, idx, 1);
    return cart; // 若沒有要修改陣列，函式就不進行複製，這有利於增加執行效率。
}
```

---

## 6.7 簡介 JavaScript 陣列 (array)

-   JavaScript 中最基本的 collection 資料型別之一。
-   元素有順序性。
-   異質的 (heterogeneous):
    一個陣列可以同時存放不同資料型別的元素。
-   JavaScript 裡的陣列可以改變長度 ↔️ C / Java 的陣列不可改變長度。

---

-   以索引存取 `idx`:
    此語法會傳回陣列中索引值為 `idx` 的元素；注意索引值從 0 開始。

```javascript
var array = [1, 2, 3, 4];
array[2];
// 3
```

-   設定元素 `idx`":
    給定索引值之後，可以用`=`指定或改變該位置的元素。

```javascript
var array = [1, 2, 3, 4];
array[2] = "abc";
array;
// [1, 2, "abc", 4]
```

---

-   陣列長度 `.length`:
    此屬性能顯示陣列的元素數量。

```javascript
var array = [1, 2, 3, 4];
array.length;
// 4
```

-   加到最後 `.push(el)`:
    將新元素（即 el）加到陣列的最末端，並傳回新的長度。

```javascript
var array = [1, 2, 3, 4];
array.push(10);
// 5
array; // [1, 2, 3, 4, 10]
```

---

-   移除末端元素 `.pop()`:
    此 method 會傳回陣列最後的元素，並將該元素移除。

```javascript
var array = [1, 2, 3, 4];
array.pop();
// 4
array; // [1, 2, 3]
```

-   加到開頭 `.unshift(el)`:
    將新元素（即 el）加到陣列開頭，並同時回傳陣列長度。

```javascript
var array = [1, 2, 3, 4];
array.unshift(10);
// 5
array; // [10, 1, 2, 3, 4]
```

---

-   移除開頭元素 `.shift()`:
    會傳回陣列開頭（索引值為 0）的元素，並將該元素移除。

```javascript
var array = [1, 2, 3, 4];
array.shift();
// 1
array; // [2, 3, 4]
```

-   複製陣列 `.slice()`:
    此 method 會建立並傳回一個陣列的`淺拷貝（shallow copy）`。

```javascript
var array = [1, 2, 3, 4];
array.slice(); // [1, 2, 3, 4]
```

---

-   移除指定元素 `.splice(idx, num)`
    此 method 會從索引 `idx` 開始，往後移除 `num` 個元素（包含 `idx` 所指的元素），並傳回移除的部分。

```javascript
var array = [1, 2, 3, 4, 5, 6];
array.splice(2, 3); // 從索引 2 開始，往後移除 3 個元素
// [3, 4, 5]
array;
// [1, 2, 6]
```

---

# 練習 6-1

---

## 6.8 如果操作既是『讀取』也是『寫入』怎麼辦？

一個函式有時需同時扮演兩種角色，即：既修改變數，又傳回值。  
這種情況在陣列的 `.shift()` method 就是個好例子：

```javascript
var a = [1, 2, 3, 4];
var b = a.shift();
console.log(b); // prints 1  -> 傳回值
console.log(a); // prints [2, 3, 4] -> 變數 a 被更改了
```

`.shift()` method 不僅會修改陣列內容，同時也會傳回陣列中的第一個元素。

---

### 要怎麼在這樣的函式中實作「寫入」時複製呢？

-> 寫入時複製的精髓就是`把「寫入」轉換成「讀取」；換言之，我們需把原本的「修改變數」變成「傳回值」`。

`.shift()` 是一個例子，它既修改陣列內容，又傳回值，設計上讓函式既讀又寫，好麻煩！🙅

---

### 有兩種方法可以處理此狀況：

1. 將函式的「讀取」和「寫入」部分拆開。
2. 讓函式回傳兩個值。

不過在你可以選擇前
請以第 1 種方法為優先，先將函式所關心的真正值寫得更清楚。

正如第 5 章所言：`設計的本質是「拆解」`。

---

## 6.9 拆解同時『讀取』與『寫入』的函式

利用寫作複製操作的兩步法共包含兩個步驟：

1. 把函式的「讀取」與「寫入」部分解開來拆成兩個函式
2. 利用「寫入時複製」將「寫入」轉化為「讀取」。

---

### 第一步：將讀取與寫入操作分開

`.shift()` method 會傳回一個「讀取」資料（即傳回值，也就是陣列中的第一個元素），並修改陣列內容。
為了拆解，我們可先寫一個只讀取的版本：

```js
function first_element(array) {
    return array[0];
    // 此函數的功能僅是傳回陣列中的第一個元素（若為空陣列則傳回 undefined）
    // 此屬於 Calculation
}
```

> 這段 `first_element()` 只會讀取，不會修改原陣列內容。

---

接著，把寫入操作也拆出來：

```js
function drop_first(array) {
    array.shift(); // 執行 .shift method , 改捨棄該 method 的傳回值
}
```

---

### 第二步：對寫入函式實作「寫入時複製」

我們已把 `.shift()` method 的「讀取」與「寫入」分開，但後者（`drop_first()` 函式）會修改傳入的陣列，因此我們要對其實作「寫入時複製」：

### 原始程式

```js
function drop_first(array) {
    array.shift();
}
```

---

### 實作寫入時複製

```js
function drop_first(array) {
    var array_copy = array.slice(); // 複製陣列
    array_copy.shift(); // 修改複製品
    return array_copy; // 傳回新陣列
    // 標準的『寫入時複製』實作
}
```

> 符合「寫入時複製」的原則！

這樣一來，讀取與寫入就被明確地拆開，你能自由選擇要不要對資料做更動。

📝 書中推薦以此方式做法（相較讓函式傳回兩個值的第二種方式）

---

## 6.10 讓一個函式傳回兩個值

第二種方法也包含兩個步驟—首先是把 `shift()` method 包裝在一個可修改的新函式中（新函式同時進行「讀取」與「寫入」），其次則是將該函式轉換為純讀取。

### 第一步：將方法包裹在函式裡

第一步，將方法包裹在一個可由我們掌控與修改的函式裡。與前面不同的是，此處不需捨棄傳回值：

```javascript
function shift(array) {
    return array.shift();
}
```

---

### 第二步：把既讀取又寫入的函式轉換成純讀取

改寫 `shift()` 函式，使其產生陣列複本、修改複本、再傳回該複本以及其中的第一個元素。以下就是實際做法：

-   原始程式

```javascript
function shift(array) {
    return array.shift();
}
```

---

-   實作寫入時複製

```javascript
function shift(array) {
    var array_copy = array.slice();
    var first = array_copy.shift();
    return {
        // 使用物件傳回
        first: first,
        array: array_copy,
    };
}
```

使用物件傳回兩個不同值

---

### 另一種選擇

可以取上一節中的兩個函式：`first_element()` 和 `drop_first()`，並把它們的傳回值放到本節 `shift()` 函式的傳回物件中：

```javascript
function shift(array) {
    return {
        first: first_element(array),
        array: drop_first(array),
    };
}
```

因為 `first_element()` 和 `drop_first()` 皆為 Calculations，所以不必再做其它改寫（上面的 `shift()` 函式也必是 Calculation）。

---

# 練習 6-2

---

## 休息一下

#### Q1：我們在第 4 章中對 `add_item()` 進行的修改實際上就是在實作寫入時複製。那麼，`add_item()` 也算是讀取函式嗎？

**A1**：是的，由於實作了寫入時複製的 `add_item()` 不會更改購物車陣列，故屬於讀取。事實上，該函式的功能就好像在問：『假如我們在此購物車中加入這項商品，陣列將會變成怎樣？』

注意！以上問題為假設性提問，而回答此類提問對思考和計劃而言非常重要。不要忘了！Calculation 函式經常用於計劃，所以將身為 Calculation 的 `add_item()` 看成假設性提問很合理。

---

#### Q2：本例的購物車是以陣列實作的。但這麼一來，為了找到給定名稱的商品，我們就必須一一走訪陣列中的元素才行。請問：陣列真的是實作購物車的最佳資料結構嗎？使用關聯資料結構（如：物件）不是更好嗎？

**A2**：使用物件的確可能更好。不過，既存程式中的變數資料結構通常早已確定，且無法輕易更改。本例正屬於這樣的狀況！我們只能在『購物車為陣列』的前提下改寫程式碼。

---

#### Q3：實作不變性好像很費功夫。這麼做真的值得嗎？有沒有更簡單的方法？

**A3**：JavaScript 中並無太多標準函式庫，所以你會感覺我們是在填寫基本功能。此外，JavaScript 也沒有支援寫入時複製，因此必須自行實作相關步驟。有鑑於此，我們有必要討論一下花那麼大功夫到底值不值得？

---

首先，如第 1 章所述，`JavaScript 並非 FP 的最佳語言，所以才需要花那麼大力氣`。假如你改用其他擁有完整函式庫、甚至是為 FP 設計的程式語言（【譯註】如本章前面提過的 Haskell 和 Clojure），則實作不變性的過程會容易許多。

其次，在 6.6 節有說過，我們可以把『寫入時複製』操作普遍化，以便用在多個地方。這麼做一開始可能很麻煩，但卻能有效降低日後操寫重複程式碼的機會。

最後，不要忘記『去除隱性輸出（如：直接寫入全域變數 shopping_cart）可大大增加函式的可測試與可重複使用性』。

---

# 練習 6-4

---

# 練習 6-5

---

## 6.11 讀取不可變資料結構屬於 Calculations

> 「我想我知道本章所說的『讀取』與『寫入』和 Action、Calculation、Data 有什麼關聯！」

> 「讀取可變資料屬於 Action，但讀取不可變資料是 Calculation。」

> 「『寫入』會改變資料。因此，只要把所有『寫入』去除，資料就不會改變！」

---

#### `讀取可變資料屬於 Actions`

若資料的值可以改變，則每次讀取的結果可能會不同，所以讀取可變資料屬於 Actions。

#### `寫入造成資料改變`

寫入操作會修改資料，故為資料可變的成因。

#### `假如一筆資料完全沒有寫入，則其必不變`

如果我們把與某資料有關的所有寫入改成讀取，則該資料的值在初始化以後就不會再改變，也就具有不變性。

---

#### `讀取不可變資料結構屬於 Calculations`

一旦我們將資料轉換為不可變，所有讀取操作就會變成 Calculations。

#### `將寫入轉為讀取，可以讓更多函式變成 Calculations`

不變的資料結構越多，程式中 Calculations 的比例越高、Actions 的比例就越低。

---

## 6.12 程式中包含隨時間而變的狀態

> 「可變資料仍是不可以的！如果購物車不可變，那買東西的程式要怎麼寫呢？」

舉例：
以 MegaMart 購物車為例，消費者該怎麼把新商品加到購物車中？

我們希望程式中一切資料皆不會變，還是應該保留一點改變的方式？

---

解法是使用 **替換（swapping）**
讓變數的內容看起來像是改變，但其實是`指向一個新值`。

### 替換步驟：

1. 讀取 -> 2. 修改 -> 3. 寫入

---

```js
// 範例：加入商品
shopping_cart = add_item(shopping_cart, shoes);
// 寫入         // 修改     // 讀取

// 範例：移除商品
shopping_cart = remove_item_by_name(shopping_cart, "shirt");
// 寫入             // 修改            // 讀取
```

總而言之，global 變數 `shopping_cart` 總是指向最新的值，而每當需要修改內容時，就應使用上述的替換程序。
在 FP 裡，`替換` 是一個既常見又強大的過程，可以讓我們輕鬆實作新增或刪除指定內容的操作。

---

## 6.13 不可變資料的效率已經夠高

> 每次要改資料前都要重建資料結構？聽起來就很沒效率？

`主要觀念`
一般而言，不可變資料會比可變資料消耗更多記憶體與效能。
但在現代電腦效能快速成長的情況下，大多數系統仍是由網路延遲而非 CPU 效能決定系統速度。

`高效能系統也能用不可變`
高速交易（HFT）系統對延遲容忍度極低，卻也能採用不可變資料設計完成。
設計良好的不可變系統，在效能與可預測性之間找到平衡。

---

`記憶體耗損化也不遜`
垃圾回收（GC）技術日益進步，使得短期產生、快速釋放的物件更易於處理。
在 FP 裡面，大多數不可變值是短生命週期的暫存資料，能被有效回收。

`複製的次數可能沒有想像那麼頻繁`
許多不可變資料結構會使用 **淺拷貝（shallow copy）** 或 **結構共用（structural sharing）**：
例如：只有列表中的某一元素改變，會共用其餘未改變部分的記憶體，避免整份資料重建。

---

### 支援 FP 的語言有更高效率的不可變資料結構

-   現代許多支援函數式編程的語言（如 Clojure）都內建高效能的不可變資料結構。
-   這些語言會共用更多記憶體、減少複製與浪費，對於垃圾回收器造成的壓力也比較低。

📝 不變性的基礎仍是寫入時複製。

---

## 6.14 作用在物件上的寫入時複製操作

在此之前的寫入時複製操作都與 JavaScript 陣列有關。但這樣還不夠！我們還需要設定購物車內商品的價格，而該資料是以物件表示的。
對物件實作寫入時複製需要設定下面相同的步驟和面相：

1. 產生複本
2. 修改複本
3. 傳回複本

---

在 JavaScript 中，陣列的淺拷貝可以透過 `.slice()` method 實現。然而，對於物件，JavaScript 標準庫中沒有一個與之直接等效的方法來複製物件。

值得注意的是，`Object.assign()` method 允許我們將一個物件中的所有可列舉屬性（鍵(key)與值(value)）複製到另一個物件中；如果目標物件是空的，這將相當於創建一個原物件的淺拷貝。下面的例子說明如何用其複製物件：

```javascript
var object = { a: 1, b: 2 };
var object_copy = Object.assign({}, object);
```

---

對 `setPrice()` 實作寫入時複製！
此函式的功能是設定商品物件的價格：

```javascript
function setPrice(item, new_price) {
    // 原始程式
    item.price = new_price;
}
```

```javascript
function setPrice(item, new_price) {
    // 寫作寫入時複製
    var item_copy = Object.assign({}, item);
    item_copy.price = new_price;
    return item_copy;
}
```

基本概念和陣列是一樣的。事實上，你可以對任意資料結構實作寫入時複製，只要依循相同的三大步驟即可。

---

## 🔖 小字典

**淺拷貝**（shallow copy）`只會複製巢狀資料中的最上層結構`。
以內含多個物件的陣列為例，淺拷貝只會複製陣列，至於其中的物件，則由原陣列與其複本共享。

兩個巢狀資料分享下層資料參照的情況稱為 `結構共享`（structural sharing）。
當一切皆不可變時，結構共享是很安全的。此技巧能降低記憶體用量，且快過複製所有資料。

---

## 6.15 簡介 JavaScript 物件

JavaScript 的物件 (object) 與其它語言裡的 hash map 或關聯陣列 (associative array) 很像。

這種資料結構由一系列`鍵 (key) / 值 (value) 配對`所組成，且鍵在同一物件中不得重複。

在一般的物件中，鍵必須為字串。([註]: 自 ES6 加入的 Map 物件中，鍵可以是任何資料型別)，值可以是任意資料型別。

---

`以鍵查值 [key]`
該操作能查找與指定鍵對應的值。如果鍵不存在，則會傳回 undefined。

```javascript
> var object = {a: 1, b: 2};
> object["a"]
1
```

`以鍵查值 .key`
你也可以使用點表示法 (.) 來存取物件的屬性。當屬性名稱符合 JavaScript 的識別符規則時，這種方法特別方便。

```javascript
> var object = {a: 1, b: 2};
> object.a
1
```

---

`設定特定鍵的值 .key 或 [key]=`
以上列的兩種寫法皆可將值指定給鍵，進而改變物件內容。
假如給定的鍵存在，則對應值會被取代；
如果不存在，則會在物件中新增鍵值對 (key-value pair)。

```javascript
> var object = {a: 1, b: 2};
> object["a"] = 7;
7
> object
{a: 7, b: 2}
> object.c = 10;
10
> object
{a: 7, b: 2, c: 10}
```

---

`刪除鍵值對 delete`

在給定鍵的情況下，delete 實行能將對應的鍵/值配對移除。
你可以在 delete 實行後面的物件加上 [key] 或 .key 來指定鍵。

```javascript
> var object = {a: 1, b: 2};
> delete object["a"];
true
> object
{b: 2}
```

---

💡`複製物件 Object.assign(a, b)`

-   該 method 比較複雜，其作用是把物件 b 的所有鍵/值配對複製到物件 a (使 a 的內容改變)。
-   倘若我們把 b 的鍵/值都複製到一空物件中，那就相當於產生了 b 的複本。

```javascript
> var object = {x: 1, y: 2};
> Object.assign({}, object);
{x: 1, y: 2}
```

---

`列出所有鍵 Object.keys()`

-   假如需要遍歷某物件中的所有鍵/值配對，可以先用 `Object.keys()` 取出該物件中所有的鍵。
-   這些鍵會以陣列的形式傳回，使我們得以用`迴圈`走訪。

```javascript
> var object = {a: 1, b: 2};
> Object.keys(object)
["a", "b"]
```

---

# 練習 6-6

---

# 練習 6-7

---

# 練習 6-8

---

# 練習 6-9

---

## 6.16 將巢狀資料的『寫入』轉換成『讀取』

任務還沒有結束！『購物車操作』中的『根據名稱更新指定商品價格』仍屬於『寫入』，而我們要將其改成『讀取』。
不過，上述操作有些特別，因為其涉及到巢狀資料結構的修改（變更的對象是購物車陣列中的商品物件）。

一般而言，轉換巢狀資料下層的操作是比較容易的。事實上，練習 6-7 已帶領各位實作了 `setPrice()`，該函式能修改商品。我們可以利用 `setPrice()` 將 `setPriceByName()`改寫成寫入時複製版本。

---

### 原始程式

```javascript
function setPriceByName(cart, name, price) {
    for (var i = 0; i < cart.length; i++) {
        if (cart[i].name === name) {
            cart[i].price = price;
        }
    }
}
```

---

### 實作寫入時複製

```javascript
function setPriceByName(cart, name, price) {
    var cartCopy = cart.slice(); // 先產生副本，再修改副本
    for (var i = 0; i < cartCopy.length; i++) {
        if (cartCopy[i].name === name) {
            // 這裡呼叫了寫入時複製的操作來更改位於巢狀結構下層的商品物件。
            cartCopy[i] = setPrice(cartCopy[i], price);
        }
    }
    return cartCopy;
}
```

---

巢狀資料的寫入時複製遵循與非巢狀相同的模式，即『產生複本、修改複本、傳回複本』。
唯一不同的是：此處得進行兩次複製：一次是複製購物車陣列，另一次則複製商品。

倘若像原始程式一樣直接更改商品物件，資料就無法維持不變。
一旦於商品在購物車陣列中的參照位置沒有變化，但其中的值卻不同了（換言之，巢狀資料上層的陣列結構相同，但下層的商品物件被改變）。
我們不能接受這樣的結果！整個巢狀資料都應保持原樣。

`巢狀結構中每一層的資料都不能改變！`
當需要修改巢狀資料時，你不僅得複製最底層的值，上面各層也得一併複製才行。

---

## 6.17 巢狀資料中的哪些都星需要複製？

假設購物車中共有三件商品：
一件 T 恤 👕、一雙鞋 👟、一對襪子 🧦。

也就是說，目前的巢狀結構包含：
一個陣列（購物車）與其下的三個物件（商品）。

目前我們想把 T 恤的價格設為 13 美元。要達成此目的，需使用能操作巢狀資料的 `setPriceByName()`，如下所示：

```javascript
shopping_cart = setPriceByName(shopping_cart, "t-shirt", 13);
```

---

我們來檢視一下完整的程式碼，看看其中進行了哪些複製：

```javascript
function setPriceByName(cart, name, price) {
    var cartCopy = cart.slice(); // 複製陣列
    for (var i = 0; i < cartCopy.length; i++) {
        if (cartCopy[i].name === name) {
            // 當遇到符合 T 恤時，會呼叫「一次」setPrice()
            cartCopy[i] = setPrice(cartCopy[i], price);
        }
    }
    return cartCopy;
}

function setPrice(item, new_price) {
    var item_copy = Object.assign({}, item); // 複製物件
    item_copy.price = new_price;
    return item_copy;
}
```

---

本例中的資料包括一個陣列與三個物件，其中有哪些被複製了呢？

答案是：
只有一個陣列（購物車）和一個物件（T 恤），另外兩個物件沒有被複製。這是怎麼一回事？

原來，此處對巢狀資料進行的複製屬於`淺拷貝`，因此產生了`結構共享`。

---

## 🔖 小字典

以下詞彙先前已介紹過，故這裡只做簡單說明：

**巢狀資料**：即資料結構內還包含另一資料結構；在內部的資料為下層，外部則為上層。

**淺拷貝**：只複製巢狀資料的上層資料結構。

**結構共享**：兩巢狀資料參照相同的內部資料結構。

---

## 6.18 將『淺拷貝』與『結構共享』視覺化

上一章的例子共包含四筆資料：一個購物車(陣列)以及三項商品(物件)。我們的任務是把 T 恤的價格設為 13 美元。

### 初始狀態

```
[陣列]
   |
   |-----> {name: "shoes", price: 10}
   |
   |-----> {name: "socks", price: 3}
   |
   |-----> {name: "t-shirt", price: 7}
```

---

程式先對購物車陣列做淺拷貝。在最一開始，複本與原陣列會指向記憶體中的相同物件。

`陣列淺拷貝後`

```
[原始陣列]                    [陣列複本]
   |                            |
   |                            |
   |-----> {name: "shoes",      |-----> {name: "shoes",
   |        price: 10}          |        price: 10}
   |                            |
   |-----> {name: "socks",      |-----> {name: "socks",
   |        price: 3}           |        price: 3}
   |                            |
   |-----> {name: "t-shirt",    |-----> {name: "t-shirt",
            price: 7}                    price: 7}
```

---

當迴圈訪問到 T 恤時，會呼叫`setPrice()`。該函式先淺拷貝 T 恤物件，再把價格改為 13。

`T 恤物件複製後`

```
[原始陣列]                    [陣列複本]
   |                            |
   |                            |
   |-----> {name: "shoes",      |-----> {name: "shoes",
   |        price: 10}          |        price: 10}
   |                            |
   |-----> {name: "socks",      |-----> {name: "socks",
   |        price: 3}           |        price: 3}
   |                            |
   |-----> {name: "t-shirt",    |-----> {name: "t-shirt",     {name: "t-shirt",
            price: 7}                    price: 7}   ------>   price: 13}
```

---

`setPrice()`把改完的物件複本傳回，傳回值在`setPriceByName()`裡被指定給陣列複本，替換原本的 T 恤物件。

### 陣列複本更新後

```
[原始陣列]                    [陣列複本]
   |                            |
   |                            |
   |-----> {name: "shoes",      |-----> {name: "shoes",
   |        price: 10}          |        price: 10}
   |                            |
   |-----> {name: "socks",      |-----> {name: "socks",
   |        price: 3}           |        price: 3}
   |                            |
   |-----> {name: "t-shirt",    |        {name: "t-shirt",
            price: 7}           |------>  price: 13}
```

---

雖然本例中有四筆資料(一個陣列與三個物件)，但只有兩筆資料被複製(陣列與其中一個物件)。
其它兩個物件因為並無變化，所以程式也沒有進行複製。

注意‼原陣列與陣列複本皆指向相同的未修改物件

此即前面所說的`結構共享` -> 只要我們不去改變共享的物件，這麼做就不會有問題。
總的來說，寫入時複製能同時保留原資料與複本，且複本上的更動不會影響原資料。

---

# 練習 ６-10

---

# 練習 ６-11

---

## 結論

-   本章說明了「寫入時複製」與讀取/寫入的關係。
-   JavaScript 的寫入時複製需要自行實作，正因為如此，將相關操作包裹在公用函式裡會比較方便，當日後需要寫入時複製時，只要使用這些包裝函式萬無一失。
-   請各位將「寫入時複製」當成教條一般遵守。

---

## 重點整理

-   FP 程式設計師喜歡不可變資料，因為使用可變資料無法寫出 Calculations。
-   在寫入時複製裡，我們會產生複本，再以修改複本來取代更動原資料。
-   實作不可變的單狀資料時，需先進行淺拷貝，然後更改複本，最後將其傳回。
-   我們可以先將基本的陣列與物件操作實作成寫入時複製函式，再利用這些函式定義更高階函
    式，這麼做可避免重複撰寫相同的程式碼。

---

## 接下來...

寫入時複製雖然強大，但別人所寫的函式裡卻未必有實作。

事實上，我們時常需要用到未實作寫入時複製的舊程式碼，此時該如何確保資料在流動過程中不被改變呢？

這就需要用到下一章介紹的技巧：

🛡️`防禦型複製 (defensive copying)`

---

# Ch7 讓不變性不受外來程式破壞

-   利用「防禦型複製」(defensive copying) 來保護你的程式，使其中的不可變資料不受其他來源的程式影響。
-   比較深拷貝與淺拷貝。
-   瞭解「防禦型複製」與「寫入時複製」的使用時機。

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
