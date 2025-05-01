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

Presenter： Yo0
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

`背景情境`
MegaMart 每週會推出一項黑色星期五促銷活動，讓行銷部門能夠快速驗證促銷活動的效果。  
門店系統中已經有處理購物車的既有程式，而且使用數年，經過多次測試、運作穩定。

> 👨‍💼:「本週黑色星期五是買三送一，我要把折扣邏輯加入購物車的程式碼，剛好又要執行結帳流程測試！」

> 👩🏼‍💻:「糟了！那段程式沒有人看得懂，這樣更動確定不會壞掉？」

---

## 🔖 小字典：  

### Legacy code（既有程式碼）

是指已經存在的程式碼，通常沒有測試，或是難以更改。
例如這段程式碼 `black_friday_promotion()` 就是 legacy code。

---

## 程式碼片段：加入商品至購物車

```javascript
function add_item_to_cart(name, price) {
    var item = make_cart_item(name, price);
    shopping_cart = add_item(shopping_cart, item);
    var total = calc_total(shopping_cart);
    set_cart_total_dom(total);
    update_shipping_icons(shopping_cart);
    update_tax_dom(total);
    black_friday_promotion(shopping_cart); // 新增的促銷行為
}
```

我們很少人能讀懂這段程式，但實質改變購物車行為的內容就藏在 `black_friday_promotion()`。

---

問題:
`black_friday_promotion()` 會破壞寫入時複製，讓函式行為不再固定可預期。

幸運的是，防禦型複製（defensive copying）可以解決這個問題。

---

## 7.2 寫入時複製函式需與未實作不變性的函式互動

`black_friday_promotion()`並沒有實作寫入時複製，所以是不受信任的函式。
此處「不受信任」不是指不安全，而是「該函式可能會修改資料」。

---

假設把所有程式碼納入下圖的兩個圓圈中，位於「安全區」內的可信任函式皆能保持資料不變，也就是說，你可以放心使用這些程式碼。

`black_friday_promotion()`並不在上述的安全區內，但我們仍一定得執行它。

執行時，客戶程式勢必會以輸入/輸出的形式與`black_friday_promotion()`交換資料。

經過整理後，所有從安全區離開的資料都可能被不受信任的函式修改，所以是潛在可變的，而從外面進入安全區者亦是如此。

不受信任的函式有可能保留資料的參照，並隨時修改參照上的值，所以在寫入時維持不變性的前提下交換資料成了一大挑戰。

---

```
                 不受信任的程式
                 ⚠️   ⚠️
              ⚠️         ⚠️
           ⚠️               ⚠️
          ⚠️    ┌───────┐    ⚠️
         ⚠️     │   ✓   │     ⚠️
         ⚠️     │ ✓   ✓ │     ⚠️  ←── 安全區
          ⚠️    └───────┘    ⚠️
           ⚠️      ↑  ↓     ⚠️
              ⚠️         ⚠️
                 ⚠️   ⚠️

← 進入安全區的資料是可變的
← 離開安全區的資料是可變的
```

---

各位已在前一章看過寫入時複製了，但該技巧在這裡卻派不上用場。寫入時複製要求在修改資料前先複製，因此你得瞭解修改發生在何處，才知道哪裡需要複製。
但在`black_friday_promotion()`的例子裡，程式碼實在太過龐雜，導致我們難以弄清該函式到底做了哪些事。
有鑑於此，此處需改用能徹底避免資料修改的強大保護措施，即防禦型複製！

---

## 7.3 防禦型複製能守護資料不變性

逃免資料被不受信任程式改變的方法是：`在資料傳入與傳出安全區時進行複製`

首先討論如何保護傳入安全區的資料。
當資料從不受信任的函式進入安全區時，應假設其為可變的。
此時需立即對其產生深拷貝（deep copy）複本，然後將源始資料丟棄。
由於這麼做能保證只有`受信任程式具有複本的參照`，故能維持資料不變。

---

O：表示原始資料 ｜ C：表示複本

### 1. 不受信任程式中的資料

```
    ⚠️
  ⚠️   ⚠️
 ⚠️     ⚠️
⚠️ ┌───┐ ⚠️
⚠️ │ ✓ │ ⚠️
⚠️ └───┘ ⚠️
 ⚠️  ↑  ⚠️
  ⚠️ O ⚠️
    ⚠️
  安全區
```

---

### 2. 資料進入安全區

```
     ⚠️
  ⚠️    ⚠️
 ⚠️      ⚠️
⚠️  ┌───┐ ⚠️
⚠️  │ ✓ │ ⚠️
⚠️  │ O │ ⚠️
 ⚠️ └───┘⚠️
  ⚠️   ⚠️
    ⚠️

原始資料O經沒用，請將其拋棄
```

---

### 3. 產生深拷貝複本

```
     ⚠️
  ⚠️    ⚠️
 ⚠️      ⚠️
⚠️  ┌───┐ ⚠️
⚠️  │ ✓ │ ⚠️
⚠️  │O C│ ⚠️
 ⚠️ └───┘⚠️
  ⚠️   ⚠️
    ⚠️

派拷貝複本留在安全區內
此資料被修改也沒關係
```

---

### 1. 安全區中的資料

```
     ⚠️
  ⚠️    ⚠️
 ⚠️      ⚠️
⚠️  ┌───┐ ⚠️
⚠️  │ ✓ │ ⚠️
⚠️  │ O │ ⚠️
 ⚠️ └───┘⚠️
  ⚠️   ⚠️
    ⚠️
  安全區
```

---

### 2. 產生深拷貝複本

```
     ⚠️
  ⚠️    ⚠️
 ⚠️      ⚠️
⚠️  ┌───┐ ⚠️
⚠️  │ ✓ │ ⚠️
⚠️  │O C│ ⚠️
 ⚠️ └───┘⚠️
  ⚠️   ⚠️
    ⚠️

原始資料從未離開安全區
```

---

### 3. 深拷貝複本離開安全區

```
     ⚠️
  ⚠️    ⚠️
 ⚠️  C   ⚠️
⚠️  ┌───┐ ⚠️
⚠️  │ ✓ │ ⚠️
⚠️  │ O │ ⚠️
 ⚠️ └───┘⚠️
  ⚠️   ⚠️
    ⚠️

派拷貝複本進入不受信任的程式
此資料被修改也沒關係
```

---

以上就是防禦型複製的概念：`當資料進入時做深拷貝，離開時也做深拷貝`。

上述操作能確保:
不可變資料永不離開安全區，可變資料則永遠無法進入。

瞭解這一點後
讓我們討論如何將此技巧套用在`black_friday_promotion()`上吧！

---

## 7.4 實作防禦型複製

什麼是防禦型複製？

當我們呼叫會改變傳入引數的函式時，防禦型複製能確保安全區中的資料維持不變，這樣就能維持程式的不變性。

例如：
    •    black_friday_promotion() 會修改 shopping_cart。
    •     解法就是在傳入前先建立 深拷貝（deep copy），確保原資料不被修改。

---

原始程式（未防禦）

```js
function add_item_to_cart(name, price) {
    var item = make_cart_item(name, price);
    shopping_cart = add_item(shopping_cart, item);

    var total = calc_total(shopping_cart);
    set_cart_total_dom(total);
    update_shipping_icons(shopping_cart);
    update_tax_dom(total);

    black_friday_promotion(shopping_cart);
}
```

問題在於 `black_friday_promotion()` 直接修改了 shopping_cart！

---

解法一：資料離開安全區前先複製

```js
function add_item_to_cart(name, price) {
    var item = make_cart_item(name, price);
    shopping_cart = add_item(shopping_cart, item);

    var total = calc_total(shopping_cart);
    set_cart_total_dom(total);
    update_shipping_icons(shopping_cart);
    update_tax_dom(total);

    var cart_copy = deepCopy(shopping_cart); // 離開安全區前先複製
    black_friday_promotion(cart_copy);
}
```

---

解法二：資料離開與進入安全區皆複製

```js
function add_item_to_cart(name, price) {
    var item = make_cart_item(name, price);
    shopping_cart = add_item(shopping_cart, item);

    var total = calc_total(shopping_cart);
    set_cart_total_dom(total);
    update_shipping_icons(shopping_cart);
    update_tax_dom(total);

    var cart_copy = deepCopy(shopping_cart); // 離開前複製
    black_friday_promotion(cart_copy);
    shopping_cart = deepCopy(cart_copy); // 進入安全區前再複製回來
}
```

---

-   防禦型複製 = 深拷貝機制（deepCopy）
-   目的：讓程式邏輯在不變性的保障下進行資料傳遞與函式呼叫
-   應用情境：當無法信任外部函式是否會改變資料時

---

## 7.5 防禦型複製的原則

當我們必須使用「未實作不變性」的程式（即：不受信任的程式）時，
防禦型複製（defensive copying） 就能確保資料維持不變。

---

## 以下是兩大基本原則：

`原則一：資料離開安全區時需複製`
當「不可變資料」要從安全區傳入「不受信任的函式」時，請依下列步驟保護資料不變性：
    1.     產生不可變資料的深拷貝副本
    2.     將複本傳入不受信任函式中

`原則二：資料進入安全區時需複製`
當從「不受信任函式」取得的資料可能已被改變，請依下列步驟保護安全區內的資料結構：
    1.     產生可能變資料的深拷貝副本
    2.     在安全區內使用複本

---

## 🔖 小字典：什麼是深拷貝？

深拷貝（deep copying）
會複製資料中從最底層到最上層的所有資料結構。

---

## 原則應用順序？

•     原則一與原則二不一定有先後順序。
•     視資料進出安全區的情境而定：
•     若要傳資料給不受信任函式 → 先離開安全區 → 適用原則一
•     若從不受信任函式取資料 → 進入安全區 → 適用原則二
•     有時需兩者同時適用。

---

## 7.6 將不受信任的程式包裝起來

背景說明

-   雖然已實作防禦型複製，但在程式中重複撰寫 deepCopy 的片段非常不方便。
-   特別是 `black_friday_promotion()` 函式會多次呼叫，若每次都手動加深拷貝，很容易出錯。

問題解法

我們可以將 `black_friday_promotion() 包裝成含有防禦型複製的函式`，提升安全性與可重用性！

---

### 撰寫防禦型複製的包裝函式

原本程式碼：

```js
function add_item_to_cart(name, price) {
    var item = make_cart_item(name, price);
    shopping_cart = add_item(shopping_cart, item);
    var total = calc_total(shopping_cart);
    set_cart_total_dom(total);
    update_shipping_icons(shopping_cart);
    update_tax_dom(total);
    var cart_copy = deepCopy(shopping_cart); // 抽出
    black_friday_promotion(cart_copy); // 抽出
    shopping_cart = deepCopy(cart_copy); //抽出
}
```

---

可以抽取成以下函式：

```js
function black_friday_promotion_safe(cart) {
    var cart_copy = deepCopy(cart);
    black_friday_promotion(cart_copy);
    return deepCopy(cart_copy);
}
```

---

修改呼叫端

將原本的：

```js
var cart_copy = deepCopy(shopping_cart);
black_friday_promotion(cart_copy);
shopping_cart = deepCopy(cart_copy);
```

改成更簡潔的：

```js
shopping_cart = black_friday_promotion_safe(shopping_cart);
```

---

擷取防禦型複製的程式

```js
function add_item_to_cart(name, price) {
    var item = make_cart_item(name, price);
    shopping_cart = add_item(shopping_cart, item);
    var total = calc_total(shopping_cart);
    set_cart_total_dom(total);
    update_shipping_icons(shopping_cart);
    update_tax_dom(total);

    // use new function
    shopping_cart = black_friday_promotion_safe(shopping_cart);
}

// new function
function black_friday_promotion_safe(cart) {
    var cart_copy = deepCopy(cart);
    black_friday_promotion(cart_copy);
    return deepCopy(cart_copy);
}
```

---

優點

-   black_friday_promotion_safe() 更容易使用。
-   不需要在呼叫處每次手動處理深拷貝。
-   能確保傳入與傳出的資料皆維持不變性。

👩🏼‍💻:「下個月同樣會用到 black_friday_promotion() 吧！只要將其包裝在具有防禦型複製的新函式裡，下次就能放心呼叫了！」

---

# 練習 7-1：封裝不受信任的函式並使用防禦型複製

MegaMart 公司使用第三方函式庫中的 `payrollCalc()` 來計算薪水，只要傳入員工紀錄的陣列，即可回傳每個人的薪資單資料。
⚠️ 注意：`payrollCalc()` 是**不受信任的函式**，可能會修改輸入資料！

`原始函式`

```js
function payrollCalc(employees) {
    return payrollChecks;
}
```

---

你的任務：請撰寫一個名為 payrollCalcSafe() 的包裝函式，使用防禦型複製技術來保護輸入與輸出資料：

```js
function payrollCalcSafe(employees) {}
```

---

包裝後安全版本

```js
function payrollCalcSafe(employees) {
    var copy = deepCopy(employees);
    var payrollChecks = payrollCalc(copy);
    return deepCopy(payrollChecks);
}
```

這樣的設計能避免 payrollCalc() 修改原始輸入 employees 陣列，
也能防止回傳資料被外部直接操作，實現雙向防禦。

---

### 練習 7-2：處理來自不受信任系統的使用者資料

MegaMart 透過一個既有系統接收使用者資料的變更（legacy system），每當有使用者更新個人設定時，就會觸發資料傳送機制，把該資料傳送至其他內部更新通知的程式中。

由於此系統是「不受信任的資料來源」，你需保障：

-   使用者資料進入「安全區」時不會直接被改變。
-   確保 processUser() 是安全區內函式。

---

## 原始程式碼如下：

```js
userChanges.subscribe(function (user) {
    processUser(user);
});
```

任務：
請根據 防禦型複製的原則 修改上方程式：
    1.     資料離開安全區時需複製
    2.     資料進入安全區時需複製

請將 processUser(user) 的使用轉為安全版本！

---

## 安全版程式碼：

```js
userChanges.subscribe(function (user) {
    var userCopy = deepCopy(user);
    processUser(userCopy);
});
```

解說：
    •     由於來自外部系統的資料 user 不可信任，因此需要先深拷貝。
    •     進入 processUser()（假設為安全區）前需進行防禦型複製。
    •     此範例只需一次 deepCopy，無需再回拷。

---

## 7.7 你或許看過的防禦型複製

防禦型複製 是一種常見但不易察覺的程式設計技巧
以下是兩個實務上的例子：

---

1. 網路應用程式開發介面中的防禦型複製

-   常見於 API（application programming interface） 的資料傳輸過程。
-   例如：當使用 JSON 形式送入 API 時，接收端會自動先將資料深拷貝成資料結構，再進行內部處理。
-   這樣的設計避免直接操作原始 JSON 對象，保證資料不被意外更改。

小結：這是一種針對資料來源與使用分離的「防禦型複製」策略，也是微服務（microservice）與服務導向架構（SOA）中的關鍵原則。

---

2. Erlang 與 Elixir 裡的防禦型複製

-   Erlang / Elixir 是函數式語言，擅長處理併發與分散式系統。
-   它們的設計理念就是「防禦型複製」的最佳實例：
-   程式間溝通是透過 mailbox 傳遞資料。
-   每個 process 的 mailbox 像是一個 queue，不會直接操作彼此的資料。
-   傳遞過程中會將資料「複製再傳送」，而不是傳參考！

優點：確保 process 間不會互相汙染彼此資料，也就自然實現了資料不變性與高可靠性。

---

## 延伸閱讀

想了解更多可參考：
    •    [Erlang 官網](https://www.erlang.org/)
    •    [Elixir 官網](https://elixir-lang.org/)

建議：若想更深入了解防禦型複製概念，學習「微服務系統」與「Erlang 的程式設計」將是很好的起點！

---

## 🔖 小字典補充：

服務導向架構（service-oriented architecture）
是一種以「服務」為核心的系統設計方式，強調模組之間資料隔離與不共享，天生支持防禦型複製。

---

## 休息一下：關於資料複製的深入思考

### <!--fit--> 問題 1：同時保留兩份資料（原始資料與複本），哪一份才代表使用者呢？

很多人會認為「使用者」物件代表某特定個體，但在函式中若存在兩份資料，可能會疑惑哪一份才是使用者？

重點是概念轉換：

-   程式中`不應該用一個物件代表一個真實個體`。
-   應理解「資料」只是與事件有關的事實記錄，並非代表特定人。
-   所以在程式設計中，「使用者資料」只是資料，不應有唯一性執念。

---

### <!--fit--> 問題 2：「寫入時複製」與「防禦型複製」看起來好像一樣，有區別嗎？

相同點：`兩者都能確保資料不變性`。

`差異在於用途：`

1. 防禦型複製：
   當資料要離開安全區，進入不受信任的函式時使用（資料跨區傳遞）。

2. 寫入時複製：
   在安全區內部為了維持不變性所採取的寫法。

---

### 技術考量

-   防禦型複製依賴「深拷貝」，而深拷貝需要遍歷資料的所有層級，代價不小。
-   若只需在安全區內局部修改資料，使用「寫入時複製」會更節省資源。

### 小結

-   寫入時複製 與 防禦型複製 各有優勢。
-   程式設計師應依據場景，選擇合適策略來維護資料不變性。

---

7.8 比較『寫入時複製』與『防禦型複製』

| 項目      | 寫入時複製                                          | 防禦型複製                                                 |
| --------- | --------------------------------------------------- | ---------------------------------------------------------- |
| **When**  | 當你能自行控制程式實作時                            | 當需與不受信任的程式交換資料時                             |
| **Where** | 安全區內（函式內部或受控環境）                      | 資料進出安全區的地方                                       |
| **Type**  | 淺拷貝，資源需求較低                                | 深拷貝，資源需求較高                                       |
| **Steps** | 1. 對欲變更資料淺拷貝<br>2. 修改複本<br>3. 傳回複本 | 1. 資料進入安全區時做深拷貝<br>2. 資料離開安全區時做深拷貝 |

---

## 7.9 深拷貝所需資源較淺拷貝高

深拷貝複本與淺拷貝複本的差異在於：

-   前者與原始資料不會共享任何結構，因為嵌狀資料內的所有物件與陣列都被複製了。
-   後者則是在沒有被修改的資料結構都是共享的：

---

```
原始資料 shopping_cart        複本             修改後的 shopping_cart
     [ , , ]  ──────────────────────────────────────────→ [ , , ]
       │ │ │                                               │ │ │
       ▼ ▼ ▼                                               ▼ │ │
  {name: "t-shirt",                         {name: "t-shirt",│ │
   price: 7}                                 price: 13}      │ │
       │                                                     │ │
       ▼                                                     │ │
  {name: "socks",      ←─────────────────────────────────────┘ │
   pric: 3}             共享參照                                │
       │                                                       │
       ▼                                                       │
  {name: "shoes",      ←───────────────────────────────────────┘
   price: 10}            共享參照
```

---

```
當原始資料來自不受信任的程式時，其中所有東西都有可能改變，所以我們得用深拷貝將每一層資料結構都複製才行：

原始資料 shopping_cart   複本      已複製 shopping_cart
     [ , , ]  ────────────────→ [ , , ]
       │ │ │                     │ │ │
       ▼ ▼ ▼                     ▼ ▼ ▼
  {name: "t-shirt",    ──→  {name: "t-shirt",
   price: 7}            複本  price: 7}
         │                         │
         ▼                         ▼
  {name: "socks",      ──→  {name: "socks",
   pric: 3}             複本  pric: 3}
          │                          │
          ▼                          ▼
  {name: "shoes",      ──→  {name: "shoes",
   price: 10}           複本  price: 10}
```

深拷貝消耗的資源較多，所以我們只會在不確定『寫入時複製』是否存在的場合中使用。

---

## 7.10 以 JavaScript 實作深拷貝很困難

深拷貝的概念很簡單，所以實作上應該也不複雜才對。
但由於 JavaScript 裡並沒有適用的標準函式庫，要正確深拷貝具實有一定難度。

雖然如何實作強深拷貝不在本書的討論範圍內，書中還是給各位一些建議。
推薦 [Lodash](lodash.com) 函式庫中的 `.cloneDeep()` 函式可產生單狀資料的深拷貝複本。
Lodash 函式庫受到眾多 JavaScript 使用者的推崇。

✏️ 這裡的『強』指能處理任何情況或資料

---

```js
function deepCopy(thing) {
    if (Array.isArray(thing)) {
        var copy = [];
        for (var i = 0; i < thing.length; i++) copy.push(deepCopy(thing[i])); // 用迴圈複製資料中的所有元素
        return copy;
    } else if (thing === null) {
        return null;
    } else if (typeof thing === "object") {
        var copy = {};
        var keys = Object.keys(thing);
        for (var i = 0; i < keys.length; i++) {
            var key = keys[i];
            copy[key] = deepCopy(thing[key]); // 用迴圈複製資料中的所有元素
        }
        return copy;
    } else {
        return thing; //字串、數字、布林值與函式本來就是不可變的，所以不需要複製
    }
}
```

---

遺憾的是，JavaScript 中還存在許多資料型別是以上函式無法應付的，
但該實作足以呈現深拷貝的關鍵，即：不僅要複製上層的陣列或物件，
還必須以遞迴走訪其中的元素。

### 實務上，建議各位務必使用諸如 Lodash 函式庫裡的強深拷貝實作。

---

# 練習 7-3

以下 5 條敘述中，有些是描述 **深拷貝（DC）**，有些是描述 **淺拷貝（SC）**。  
請在適用於深拷貝者前標上 DC，適用於淺拷貝者則標 SC。

1. 此類拷貝會複製與資料中的每一層結構。
2. 此類拷貝允許結構共享，因此較另一種更有效率。
3. 此類拷貝會在結構共享的情況下修改元素。
4. 當執行在無結構共享時，此類拷貝能保護來自不受信任程式的資料。
5. 可利用此類拷貝實作 **無共享架構**（shared nothing architecture）。

---

1. **DC**：此類拷貝會複製與資料中的每一層結構。
2. **SC**：此類拷貝允許結構共享，因此較另一種更有效率。
3. **SC**：此類拷貝會在結構共享的情況下修改元素。
4. **DC**：當執行在無結構共享時，此類拷貝能保護來自不受信任程式的資料。
5. **DC**：可利用此類拷貝實作 **無共享架構**（shared nothing architecture）。

---

## ７.11 想像『寫入時複製』與『防禦型複製』之間的對話

✍️ 寫入時複製：

我保證資料不變，所以當然比較重要啦！😎

🛡️ 防禦型複製：

不對吧！我也能讓資料不變啊！🤨

---

✍️ 寫入時複製：

但我用的淺拷貝更快，我的效率可是好上好多倍喔～ ⚡️

🛡️ 防禦型複製：

效率你來說不見得最重要喔！重點是只要資料一離開或進入安全區，我就能確保它不會被改變！🧠

---

✍️ 寫入時複製：

安全區的設計完全是為了讓我可以發揮啊～這才是我存在的意義！💼

🛡️ 防禦型複製：

理想很好，但你也知道實務上那些 API、外部函式、黑箱太多了啦，哪有可能全部改寫？現實是殘酷的！💻❌

---

✍️ 寫入時複製：

你說得對……我不能沒有你！我們是好夥伴！😭

🛡️ 防禦型複製（眼眶泛淚）：

別哭啦！我也離不開你啊……🥹

> 於是，他們停止爭辯，並緊緊地擁抱在一起……

---

# 練習 7-4

以下 10 條敘述中，有些是描述防禦型複製（Defensive Copying, DC），
有些則是描述寫入時複製（Copy-on-Write, CW），
請在每句敘述前標上 `DC` 或 `CW`：

---

1. 此技巧依賴深拷貝。
2. 相較另一種做法，此技巧所需的資源較少。
3. 此技巧對維持資料不變性來說非常重要。
4. 此技巧會在修改資料前先產生複本。
5. 所傳用此技巧處理安全區內的資料不變。
6. 當與不受信任程式交換資料時，應使用此技巧。
7. 此技巧依賴淺拷貝。
8. 此技巧可以安全地取代另一種做法。
9. 傳送資料給不受信任的函式時，需先複製資料。
10. 接受不受信任程式送來的資料時，需複製。

---

1. **DC**：此技巧依賴深拷貝。
2. **CW**：相較另一種做法，此技巧所需的資源較少。
3. **DC 和 CW**：此技巧對維持資料不變性來說非常重要。
4. **CW**：此技巧會在修改資料前先產生複本。
5. **CW**：所傳用此技巧處理安全區內的資料不變。
6. **DC**：當與不受信任程式交換資料時，應使用此技巧。
7. **DC**：此技巧依賴淺拷貝。
8. **CW**：此技巧可以安全地取代另一種做法。
9. **DC**：傳送資料給不受信任的函式時，需先複製資料。
10. **DC**：接受不受信任程式送來的資料時，需複製。

---

## 練習 7-5

現在，有一項新任務要求你的程式與既有函式互動，但該函式可能會改變資料。選出所有適當的選項，並解釋原因。請問：你應該採取以下哪些措施來維持不變性呢？選出所有適當的選項，並解釋原因。

1. 與既有函式交換資料時使用防禦型複製。
2. 與既有函式交換資料時使用寫入時複製。
3. 實際閱讀既有函式的程式碼，看其是否會改變資料。假如不會，那就不需要做任何事。
4. 以寫入時複製重寫既有函式，然後直接呼叫新函式。
5. 本例中的既有函式也是你的開發團隊所寫，所以理所當然值得信任。

---

## 練習 7-5 解答

1. ✅！防禦型複製雖然會消耗記憶體資源以產生複本，但確實可以保護安全區。
2. ❌！只有當呼叫的函式有實作寫入時複製時，此方法才行得通。如果你無法確定既有函式如何實作，請不要假設其中包含寫入時複製。
3. 依情況而定！
   檢視原始碼的確能幫助我們瞭解既有函式是否有更動資料。但要特別注意該函式是否還做了其它事情，例如：把資料再傳給第三方程式。
4. ✅！倘若時間足夠，以寫入時複製改寫確實能解決問題。
5. ❌！就算既有函式來自你的團隊，也不應該預設它實作了資料不變性。

---

## 結論

在本章中，我們學會了兩種強大的資料不變性實作方法：

1. 寫入時複製（Copy-on-write）：
   在程式邏輯可控時，效率高、淺拷貝即可應用。
2. 防禦型複製（Defensive Copying）：
   當資料要離開安全區、進入不可預期或不受信任的函式時，能有效保護資料不被改變。

小提醒：不要把防禦型複製當作寫入時複製的替代，而是根據「場景」靈活選擇，甚至可以搭配使用！

---

重點整理

-   防禦型複製是`針對跨安全區存取資料時保護資料`的一種方法。
-   因為深拷貝會耗費更多資源，所以只在必要時使用。
-   防禦型複製確保與不變性程式互動時，資料仍可保持不變。
-   寫入時複製則拷貝較少資料，當資料修改只發生在受控區域時更有效率。
-   深拷貝會遍歷每一層資料結構，成本高，但也最穩定。

---

# Thank You For Your Listening 🥰

接下來…
下一章，我們將以本章知識為基礎，討論一種可以改變系統設計思維的程式架構！

-   Presenter : Hannah
-   Note Taker : Monica
    Date : 2025/05/15
