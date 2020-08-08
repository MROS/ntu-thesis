## 嘗試以最小堆積（min heap）構造持久化 LRU

我們也可以巧妙地用雜湊搭配最小堆積（min heap）來實作持久化 LRU 。

在這種構造中，雜湊表記錄了`賬戶地址 -> (時間戳, 賬戶資訊)`，而堆積會用單向指標來實作，堆積中每一個節點會儲存賬戶地址以及時間戳。

以下是堆積中一個節點的結構，在我們所維護最小堆積中，父節點的時間戳都小於自己的時間戳。

``` cpp
struct Node<Key> {
    Node *left;           // 左子節點
    Node *right;          // 右子節點
    Key key;              // key 在此應用中即爲賬戶地址
    TimeStamp timestamp;  // 時間戳，可用整數作爲 TimeStamp ，例如之前可能有 using TimeStamp = int;
};
```

時間戳擁有「只增不減」的特性，在 LRU 應用中的最小堆積的節點的時間戳只會變大，因此節點只會下移（shiftdown），不會發生上移（shiftup），也因此並不需要記錄父節點的指標。

在堆積加入新元素的時候，也有可能發生上移，但快取的大小是有限的，上移發生的次數也是有限的，我們可以採用特殊的初始化方式來規避上移。

下移的虛擬碼如下：

``` cpp
void shiftdown(Node *node) {
    // 若沒有子節點，結束
    if (node->left == NULL && node->right == NULL) {
        return;
    }

    // 若有子節點，找出時間戳較小的
    Node *small_child = node->left;
    if (node->right != NULL && node->right->timestamp < small_child->timestamp) {
        small_child = node->right;
    }
    // 交換父子節點
    swap(node->key, small_child->key);
    swap(node->timestamp, small_child->timestamp);
    // 向下遞迴
    shiftdown(small_child);
}
```

雜湊表則可表示成

``` cpp
struct Info<Value> {
    TimeStamp timestamp;
    Value value;           // Value 爲賬戶狀態
}
map<Key, Info> table;
```

### 快取已滿

#### 維護條件

我們考慮快取已滿時的狀態，假設當前的堆積符合最小堆的規則，亦即，除了根節點以外的所有節點都滿足

條件一、 父節點的時間戳都小於自己的時間戳

除此之外，我們還要維護另一個條件

條件二、 堆積的根的時間戳與雜湊表中的時間戳相同

解釋一下上面這個條件：雜湊表中我們可以由地址去查詢到時間戳，而在堆積中的每個節點，地址也都對應到一個時間戳。然而在整個 LRU 進行操作的過程中，除了堆積的根的時間戳，雜湊表中的時間戳跟堆積中的時間戳並不總是同步的，這並不要緊，只要保證堆積的根的時間戳與雜湊表中的時間戳同步，我們就能夠順利完成 LRU 的操作。

#### LRU get

若快取沒有命中，不做任何事；若快取命中，更新雜湊表中的時間序。

寫爲虛擬碼如下：
``` cpp
if (table.has(address)) {   // 如果雜湊表中含有所求地址
    Info info = table.get(address);
    info.timestamp = current_timestamp++;
    table.set(address, info);
    maintain_root();
}
```

在以上過程中，我們只更新了雜湊表中的時間戳，因此雜湊表中的時間戳在 get 之後就會與堆積中的時間戳不同，在 get 的地址非根時無所謂，但若恰巧 get 到了堆積的根對應的地址，我們就得執行一個額外操作來維護前一節所提到的條件二。

這個額外操作的虛擬碼如下：

``` cpp
void maintain_root() {
    while (heap.root.timestamp != table[heap.root.key].timestamp) {
        heap.root.timestamp = table[heap.root.key].timestamp;
        shiftdown(heap.root);
    }
}
```

#### LRU put

若快取沒有命中，我們得丟棄最舊賬戶，並加入新賬戶。具體操作會將時間戳最小的節點，亦即堆積的根（由於條件二，堆積的根總是最舊的）改爲欲置入的新賬戶，並執行 maintain\_root 來維護堆積規則。

若快取命中，更新雜湊表中的時間序跟賬戶狀態，並注意是否更動到堆積的根。

虛擬碼如下

``` cpp
void lru_put(Key address, Value value) {
    if (table.has(address)) {   // 快取命中
        Info info = table.get(address);
        info.timestamp = current_timestamp++;
        info.value = value;
        table.set(address, info);
        maintain_root();
    } else {                    // 快取失效，丟棄最舊賬戶並塞入新賬戶資訊
        table.remove(heap.root.key);
        table.set(address, { current_timestamp, value });
        heap.root.key = address;
        heap.root.timestamp = current_timestamp++;
        shiftdown(heap.root);
        maintain_root();
    }
}
```

### 時空間複雜度分析
### 如何填滿快取