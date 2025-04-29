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

-   「我想我知道本章所說的『讀取』與『寫入』和 Action、Calculation、Data 有什麼關聯！」
-   「讀取可變資料屬於 Action，但讀取不可變資料是 Calculation。」
-   「『寫入』會改變資料。因此，只要把所有『寫入』去除，資料就不會改變！」

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
