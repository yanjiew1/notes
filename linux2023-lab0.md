# 2023q1 Homework1 (lab0)
contributed by < `yanjiew1` >

## 開發環境

```bash
$ gcc --version
gcc (Ubuntu 11.3.0-1ubuntu1~22.04) 11.3.0

$ lscpu
Architecture:            x86_64
  CPU op-mode(s):        32-bit, 64-bit
  Address sizes:         39 bits physical, 48 bits virtual
  Byte Order:            Little Endian
CPU(s):                  8
  On-line CPU(s) list:   0-7
Vendor ID:               GenuineIntel
  Model name:            Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz
    CPU family:          6
    Model:               142
    Thread(s) per core:  2
    Core(s) per socket:  4
    Socket(s):           1
    Stepping:            10
    CPU max MHz:         3400.0000
    CPU min MHz:         400.0000
    BogoMIPS:            3600.00
```

## 目前成果

目前 `make test` 分數是 95 分，剩下 constant time checking 尚未通過。

[GitHub 網址](https://github.com/yanjiew1/lab0-c)
[GitHub Action](https://github.com/yanjiew1/lab0-c/actions/runs/4202342163)

## 待辦清單
- [x] 補完 `queue.c` 實作共筆
- [ ] 研究 Valgrind 和自動測試程式
- [ ] 改善 web server (用 `select` 或 `epoll`)
- [ ] 亂數、常數執行時間檢測、論文閱讀
- [ ] 研究 Linux Kernel 的排序程式
- [ ] 改進 merge K-list 效能。
- [ ] 共筆寫上更詳細的開發過程紀錄

## 開發過程

要 commit 時發現在 `report.c` 中的 `static char *msg_name_text[N_MSG]` 變數宣告會觸發 cppcheck 錯誤。原本打算把此行改為 `static char * const msg_name_text[N_MSG]` ，但後來發現 GitHub 有更新新的程式碼加入 `// cppcheck-suppress constVariable` 修正此問題，因此我直接在 GitHub 網頁介面上操作 sync fork 取得更新的程式至自己 fork 出來的 repository 。

因為自己只寫了一點點程式，故直接捨棄本地端的 commit ，用 GitHub 上新的程式覆寫本地的 repository 。
:::warning
可善用 `git rebase`
:notes: jserv
:::

:::info
謝謝老師，我會嘗試練習 `git rebase`
:::

## 完成 `queue.c` 實作

### `q_new`

透過 `malloc` 來配置記憶體，使用 `list.h` 裡面提供的 `INIT_LIST_HEAD` 來初始化 `struct list_head` 。

Commit: [0f09bf6](https://github.com/yanjiew1/lab0-c/commit/0f09bf6ab87795c13f8aae2e6f773e48ce41eb0a)

```c
/* Create an empty queue */
struct list_head *q_new()
{
    struct list_head *new = malloc(sizeof(struct list_head));
    if (!new)
        return NULL;

    INIT_LIST_HEAD(new);

    return new;
}
```


### `q_free`


使用 `list.h` 內提供的 `list_for_each_entry_safe` 來走訪每一個元素。使用的過程中，`safe` 會存放下一個節點，而 `it` 節點可以從 linked list 移除，不影響 `list_for_each_entry_safe` 運作。故在這裡，可以直接在走訪的過程中釋放每一個節點。最後再呼叫 `free` 把 linked list 本身釋放。

```c
/* Free all storage used by queue */
void q_free(struct list_head *l)
{
    if (!l)
        return;

    element_t *safe, *it;
    list_for_each_entry_safe (it, safe, l, list)
        q_release_element(it);

    free(l);
}
```

#### 開發過程記錄

一開始程式碼如下：

```c
/* Free all storage used by queue */
void q_free(struct list_head *l)
{
    struct list_head *safe;
    struct list_head *it;

    list_for_each_safe (it, safe, l) {
        list_del(it);
        free(it);
    }

    free(l);
}
```

隨後很快發現，應該要釋放 `element_t` 才對，因為 `struct list_head` 是包含在 `element_t` 裡面的一個成員。故改成如下結果：

```c
/* Free all storage used by queue */
void q_free(struct list_head *l)
{
    struct list_head *safe;
    struct list_head *it;

    list_for_each_safe (it, safe, l) {
        element_t *elem = list_entry(it, element_t, list);
        list_del(it);
        free(elem->value);
        free(elem);
    }

    free(l);
}
```

但因為釋放 `element_t` 很常被用到，所以後來我寫一個獨立函式 `q_free_elem` 來做。最後又發現 `queue.h` 有提供 `q_release_element` ，最後改成用 `q_release_element` ，程式如下：

```c
/* Free all storage used by queue */
void q_free(struct list_head *l)
{
    element_t *safe;
    element_t *it;

    list_for_each_entry_safe (it, safe, l, list) {
        list_del(&it->list);
        q_release_element(it);
    }

    free(l);
}
```

後來又仔細研究 `list_for_each_entry_safe` 巨集後，發現 `list_del` 可以不用做。此巨集的特性是我們可以把 `it` 直接釋放，只要不要動到 `safe` ，就可以安全操作。最後改為如下：
```c
/* Free all storage used by queue */
void q_free(struct list_head *l)
{
    if (!l)
        return;

    element_t *safe, *it;
    list_for_each_entry_safe (it, safe, l, list)
        q_release_element(it);

    free(l);
}
```


### `q_new_elem`

配合 `q_insert_head` 和 `q_insert_tail` 要新增元素，新增此函式，用來建立 `element_t` 。

使用 `malloc` 來配置記憶體空間。使用 `strdup` 來配置字串空間並拷貝字串。過程中如果配置失敗，則回傳 `NULL` 。

commit [1e56bd7](https://github.com/yanjiew1/lab0-c/commit/1e56bd7b5c02c5dcc2710b7ba90f3ccd294814c9)

```c
/* Create a new element with the provided string */
static inline element_t *q_new_elem(char *s)
{
    element_t *elem = malloc(sizeof(element_t));
    if (!elem)
        return NULL;

    char *tmp = strdup(s);
    if (!tmp) {
        free(elem);
        return NULL;
    }

    elem->value = tmp;
    return elem;
}
```


### `q_insert_head` 和 `q_insert_tail`

一開始要檢查 `head` 是否為 `NULL` ，若為 `NULL` 則不進行任何操作。

使用前面建立的 `q_new_elem` 來建立 `element_t` 。並且透過 `list_add` 或 `list_add_tail` 來把節點串上去。 若過程中失敗則回傳 `false` 。

```c
/* Insert an element at head of queue */
bool q_insert_head(struct list_head *head, char *s)
{
    if (!head)
        return false;

    element_t *elem = q_new_elem(s);
    if (!elem)
        return false;

    list_add(&elem->list, head);

    return true;
}

/* Insert an element at tail of queue */
bool q_insert_tail(struct list_head *head, char *s)
{
    if (!head)
        return false;

    element_t *elem = q_new_elem(s);
    if (!elem)
        return false;

    list_add_tail(&elem->list, head);

    return true;
}
```

### `q_copy_string`

在 `q_remove_head` 和 `q_remove_tail` 會需要拷貝字串，且 buffer 的大小是有限制的。 C 語言提供的 `strcpy` 無法滿足此需求，故在此實作另一個函數 `q_copy_string` 來完成字串拷貝。

```c
static inline void q_copy_string(char *dest, size_t size, const char *src)
{
    size_t i;
    for (i = 0; i < size - 1 && src[i] != '\0'; i++)
        dest[i] = src[i];

    dest[i] = '\0';
}
```

### `q_remove_head` 和 `q_remove_tail`

這裡實作從 queue 中移除頭或尾。

利用 `list_first_entry` 和 `list_last_entry` 來取得要移除的元素。使用 `list_del` 來移除元素。題目要求要把移除元素的字串值拷貝到 `sp` 這個 buffer ，並且 buffer 大小為 `bufsize` ，就透過前面實作的 `q_copy_string` 來做。

```c
/* Remove an element from head of queue */
element_t *q_remove_head(struct list_head *head, char *sp, size_t bufsize)
{
    if (!head || list_empty(head))
        return NULL;

    element_t *elem = list_first_entry(head, element_t, list);
    list_del(&elem->list);

    if (sp && bufsize)
        q_copy_string(sp, bufsize, elem->value);

    return elem;
}

/* Remove an element from tail of queue */
element_t *q_remove_tail(struct list_head *head, char *sp, size_t bufsize)
{
    if (!head || list_empty(head))
        return NULL;

    element_t *elem = list_last_entry(head, element_t, list);
    list_del(&elem->list);

    if (sp && bufsize)
        q_copy_string(sp, bufsize, elem->value);

    return elem;
}
```

### `q_size`

直接使用 `list.h` 提供的 `list_for_each` 來走訪每一個節點。每走訪一個就把 `count` 變數加1，最後再回傳 `count` 變數的值。

```c
/* Return number of elements in queue */
int q_size(struct list_head *head)
{
    if (!head)
        return 0;

    int count = 0;
    struct list_head *it;

    list_for_each (it, head)
        count++;

    return count;
}
```


### `q_delete_mid`

這裡充份利用雙向鏈結串列的特性，從頭尾開始走訪節點，直到二個指標碰面時，即取得中間的節點。最後再把中間節點所代表的元素刪除。

```c
/* Delete the middle node in queue */
bool q_delete_mid(struct list_head *head)
{
    // https://leetcode.com/problems/delete-the-middle-node-of-a-linked-list/
    if (!head || list_empty(head))
        return false;

    struct list_head *left = head->next;
    struct list_head *right = head->prev;

    while (left != right && left->next != right) {
        left = left->next;
        right = right->prev;
    }

    list_del(right);
    element_t *elem = list_entry(right, element_t, list);
    q_release_element(elem);

    return true;
}
```

### `q_delete_dup`

我的作法是先宣告 `cut` 變數，其內容是存放目前走訪元素往前看，第一個其元素值與目前元素不同的元素。此外另外宣告一個 `trash` 作為垃圾筒，要丟掉的元素都會先放在這裡，最後再一起清掉。

在走訪元素的過程中， `&safe->list != head && strcmp(safe->value, it->value) == 0` 這裡會去判斷下一個元素跟目前元素的值是否相同。若相同時，就繼續走訪，直到遇到下個元素與目前元素值不同時，才進行接下來的動作。

當下個元素與目前元素不同時，會先去檢查 `cut` 變數，是不是指向前一個元素。若是指向前一個元素，代表目前元素跟前一個元素不同，則不用處置。若 `cut` 不是指向前一個元素，則代表 `cut` 下一個元素到目前的元素其值均相同，故把 (`cut`, `it`] 這中間的元素全部丟到 `trash` 中。

迴圈最後，因為目前元素與下一個元素值不同，故更新 `cut` 為目前元素後，再進行下個迴圈。

把元素都丟到 `trash` 後，要清除 `trash` 裡的元素。這裡用 `list_for_each_entry_safe` 來走訪每一個 `trash` 中的元素，並用 `q_release_element` 來刪除它。

```c
/* Delete all nodes that have duplicate string */
bool q_delete_dup(struct list_head *head)
{
    // https://leetcode.com/problems/remove-duplicates-from-sorted-list-ii/
    if (!head)
        return false;

    LIST_HEAD(trash);
    element_t *it, *safe;
    struct list_head *cut = head;

    list_for_each_entry_safe (it, safe, head, list) {
        if (&safe->list != head && strcmp(safe->value, it->value) == 0)
            continue;
        /* Detect duplicated elements */
        if (it->list.prev != cut) {
            LIST_HEAD(tmp);
            list_cut_position(&tmp, cut, &it->list);
            list_splice(&tmp, &trash);
        }
        cut = safe->list.prev;
    }

    /* empty trash */
    list_for_each_entry_safe (it, safe, &trash, list)
        q_release_element(it);

    return true;
}
```

### `q_reverse`

這裡的實作很簡單，就是用 `list_for_each_safe` 走訪每一個節點。走訪過程中，用 `list_move` 把每一個節點移到開頭，這樣子順序就反過來了。 `list_for_each_safe` 允許對目前走訪的節點移動位置。因為是往前移到開頭，故不會因為節點移動而改變走訪次序或是重複走訪。

```c
/* Reverse elements in queue */
void q_reverse(struct list_head *head)
{
    if (!head)
        return;

    struct list_head *it, *safe;
    /* Iterate the list and move each item to the head */
    list_for_each_safe (it, safe, head) {
        list_move(it, head);
    }
}
```

### `q_reverseK`

這裡使用 `cnt` 來記錄已經走訪了幾個節點，一開始把 `cnt` 設為 `k` ，每走訪一個節點就把 `cnt` 減去 1 。當走訪 `k` 個節點時 (`--cnt` 變成 0)，就會從 `cut` 記錄的切點切到目前走訪的節點(不包含 `cut` 但包含 `it`，即 `(cut, it]`)切下這一串放到 `tmp` 上，再重用前面實作好的 `q_reverse` ，對 `tmp` 做反轉，再用 `list_splice` 把 `tmp` 接回來。

最後設定 `safe->prev` 為新的 `cut` 。不用 `it` 是因為 `it` 已在反轉的過程中移動位置了。

```c
/* Reverse the nodes of the list k at a time */
void q_reverseK(struct list_head *head, int k)
{
    // https://leetcode.com/problems/reverse-nodes-in-k-group/
    if (!head)
        return;
    struct list_head *it, *safe, *cut;
    int cnt = k;
    cut = head;
    list_for_each_safe (it, safe, head) {
        if (--cnt)
            continue;
        LIST_HEAD(tmp);
        cnt = k;
        list_cut_position(&tmp, cut, it);
        q_reverse(&tmp);
        list_splice(&tmp, cut);
        cut = safe->prev;
    }
}
```

### `q_swap`

`swap` 就是二個二個一組進行 `reverse` ，故直接重用 `q_reverseK` 的程式 。

```c
/* Swap every two adjacent nodes */
void q_swap(struct list_head *head)
{
    // https://leetcode.com/problems/swap-nodes-in-pairs/
    q_reverseK(head, 2);
}

```

### `q_sort`

採用 merge sort 遞迴演算法。一開始運用雙向鏈結串列的特性，從頭尾開始往中間找到中間節點，找到後，分別把 (`head`, `mid`] 和 (`mid`, `head`) 切下來加到 `left` 、 `right` 二個臨時鏈結串列。再針對 `left` 、 `right` 遞迴呼叫 `q_sort` ，對子串列進行排序。最後再把排序好的 `left` 和 `right` 合併，並接回 `head` 串列。

:::info
**TODO** 研究 Linux Kernel 的排序程式並試圖把 kernel 排序程式移殖過來。
:::

```c
/* Sort elements of queue in ascending order */
void q_sort(struct list_head *head)
{
    /* Try to use merge sort*/
    if (!head || list_empty(head) || list_is_singular(head))
        return;

    /* Find middle point */
    struct list_head *mid;
    {
        struct list_head *left, *right;
        left = head->next;
        right = head->prev;

        while (left != right && left->next != right) {
            left = left->next;
            right = right->prev;
        }
        mid = left;
    }

    /* Divide into two part */
    LIST_HEAD(left);
    LIST_HEAD(right);

    list_cut_position(&left, head, mid);
    list_splice_init(head, &right);

    /* Conquer */
    q_sort(&left);
    q_sort(&right);

    /* Merge */
    while (!list_empty(&left) && !list_empty(&right)) {
        if (strcmp(list_first_entry(&left, element_t, list)->value,
                   list_first_entry(&right, element_t, list)->value) <= 0) {
            list_move_tail(left.next, head);
        } else {
            list_move_tail(right.next, head);
        }
    }

    list_splice_tail(&left, head);
    list_splice_tail(&right, head);
}
```

### `q_descend`

我的想法就是從尾到頭掃過一次。在掃的過程中，會判斷下一個節點(`prev`)是否小於等於目前節點(`cur`)，若是就把下一個節點(`prev`)刪除再重複迴圈，直到下一個節點是大於目前節點時，才會把 `cur` 移動到下一個節點，並且 `cnt` (計數器) 加 1 。

`list.h` 並沒有把 Linux kernel 裡 `include/linux/list.h` 中的 `list_for_each_prev` 放進來。若有它則可以用它來實作。

```c
/* Remove every node which has a node with a strictly greater value anywhere to
 * the right side of it */
int q_descend(struct list_head *head)
{
    // https://leetcode.com/problems/remove-nodes-from-linked-list/
    if (!head || list_empty(head))
        return 0;

    /**
     * Traverse from the last entry and remove the element that is
     * smaller or equal to its right. Also count the number of elements.
     */
    int cnt = 1;
    element_t *cur = list_last_entry(head, element_t, list);
    while (cur->list.prev != head) {
        element_t *prev = list_last_entry(&cur->list, element_t, list);
        if (strcmp(prev->value, cur->value) <= 0) {
            list_del(&prev->list);
            q_release_element(prev);
        } else {
            cnt++;
            cur = prev;
        }
    }

    return cnt;
}
```

### `q_merge`

這裡重用前面實作的 `q_sort` 。我把所有鏈結串列接在一起，然後呼叫 `q_sort` 排序好，放回第一個串列 。這裡很顯然沒有應用到各個子串列原本就已經排序好的特性，所以還有改善空間。但目前的作法已可達到 $O(n\ log\ n)$ 的時間複雜度。

:::info
**TODO** 善用各個子串列已排序好的特性來實作
:::

```c
/* Merge all the queues into one sorted queue, which is in ascending order */
int q_merge(struct list_head *head)
{
    // https://leetcode.com/problems/merge-k-sorted-lists/
    if (!head)
        return 0;

    /**
     * Not quite optimized but can be done in O(nlogn)
     * It can be improve later.
     */
    LIST_HEAD(tmp);
    queue_contex_t *it;
    /**
     * The macro, list_for_each_entry, exists,
     * but cppcheck tells me it is unknown
     */
    // cppcheck-suppress unknownMacro
    list_for_each_entry (it, head, chain)
        list_splice_init(it->q, &tmp);

    int size = q_size(&tmp);
    q_sort(&tmp);
    list_splice(&tmp, list_first_entry(head, queue_contex_t, chain)->q);

    return size;
}
```

## Valgrind 與 Address Sanitizer 記憶體檢查

### Makefile 中關於 `make valgrind` 的內容分析

```
valgrind: valgrind_existence
	# Explicitly disable sanitizer(s)
	$(MAKE) clean SANITIZER=0 qtest
	$(eval patched_file := $(shell mktemp /tmp/qtest.XXXXXX))
	cp qtest $(patched_file)
	chmod u+x $(patched_file)
	sed -i "s/alarm/isnan/g" $(patched_file)
	scripts/driver.py -p $(patched_file) --valgrind $(TCASE)
	@echo
	@echo "Test with specific case by running command:" 
	@echo "scripts/driver.py -p $(patched_file) --valgrind -t <tid>"
```

裡面看到很神奇的部份，就是在用 Valgrind 執行時，會把 `qtest` 執行檔中，所有 `alarm` 改為 `isnan` 。我的猜測是因為 Valgrind 會使程式執行速度變慢，導致一些操作會超過時間限制。我把這行拿掉 `sed -i "s/alarm/isnan/g" $(patched_file)` ，執行 `make valgrind` ，果然就出現超時訊息。查了一下 C 函式庫，`isnan` 是用來檢查傳入的 floating point 數值是否為 NaN ，會選用這個函數來取代 `alarm` ，估計是因為它跟 `alarm` 一樣函數名稱都是 5 個字元，此外 `isnan` 沒有任何 side effects ，故剛好可以拿來用。為了測試，我嘗試把 `isnan` 替換成 `asinf` 看看能不能運作。實測的結果如預期，可以正常運作。

### 使用 Valgrind

執行 `make valgrind` ，產生了很多 Memory leak 的訊息。以下為節錄的訊息：

```
+++ TESTING trace trace-01-ops:
# Test of insert_head and remove_head
==30743== 32 bytes in 1 blocks are still reachable in loss record 1 of 2
==30743==    at 0x4848899: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==30743==    by 0x10CBE6: do_new (qtest.c:146)
==30743==    by 0x10DE12: interpret_cmda (console.c:181)
==30743==    by 0x10E3C7: interpret_cmd (console.c:201)
==30743==    by 0x10E7C8: cmd_select (console.c:610)
==30743==    by 0x10F0B4: run_console (console.c:705)
==30743==    by 0x10D204: main (qtest.c:1228)
==30743== 
```

我先用 `valgrind ./qtest` 的方式測試，發現只要在結束前沒有把所有的 queue 釋放，就會產生上述訊息。後來發現在 `q_quit` 裡面，在第一行是 `return true;` 使下面釋放記憶體的程式沒辦法執行。把這行拿掉後就正常了。

```diff
--- a/qtest.c
+++ b/qtest.c
@@ -1059,7 +1059,6 @@ static void q_init()
 
 static bool q_quit(int argc, char *argv[])
 {
-    return true;
     report(3, "Freeing queue");
     if (current && current->size > BIG_LIST_SIZE)
         set_cautious_mode(false);
```

但若是使用 `valgrind ./qtest < traces/trace-01-ops.cmd` 的方式來執行，仍然會出現下面訊息：

```
==33817== 130 bytes in 10 blocks are still reachable in loss record 1 of 3
==33817==    at 0x4848899: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==33817==    by 0x49FE60E: strdup (strdup.c:42)
==33817==    by 0x1121B9: line_history_add (linenoise.c:1275)
==33817==    by 0x113014: line_hostory_load (linenoise.c:1360)
==33817==    by 0x10D332: main (qtest.c:1215)
==33817== 
==33817== 130 bytes in 10 blocks are still reachable in loss record 2 of 3
==33817==    at 0x4848899: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==33817==    by 0x49FE60E: strdup (strdup.c:42)
==33817==    by 0x1121B9: line_history_add (linenoise.c:1275)
==33817==    by 0x10F10C: run_console (console.c:692)
==33817==    by 0x10D2D7: main (qtest.c:1227)
==33817== 
==33817== 160 bytes in 1 blocks are still reachable in loss record 3 of 3
==33817==    at 0x4848899: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==33817==    by 0x112205: line_history_add (linenoise.c:1263)
==33817==    by 0x113014: line_hostory_load (linenoise.c:1360)
==33817==    by 0x10D332: main (qtest.c:1215)
==33817== 
```

從上述訊息可以看出是 linenoise 中處理 history 的部份出錯。分析了一下 linenoise.c 的程式碼，發現 `line_atexit` 只會在 `enable_raw_mode` 函式裡被註冊為 atexit function ，但是在 stdin 不為 tty 的情況下 `enable_raw_mode` 不會被呼叫到。故把註冊 `line_atexit` 的程式移到 `linenoise` function 就解決了。

```diff
--- a/linenoise.c
+++ b/linenoise.c
@@ -243,10 +243,6 @@ static int enable_raw_mode(int fd)
 {
     if (!isatty(STDIN_FILENO))
         goto fatal;
-    if (!atexit_registered) {
-        atexit(line_atexit);
-        atexit_registered = true;
-    }
     if (tcgetattr(fd, &orig_termios) == -1)
         goto fatal;
 
@@ -1189,6 +1185,11 @@ char *linenoise(const char *prompt)
 {
     char buf[LINENOISE_MAX_LINE];
 
+    if (!atexit_registered) {
+        atexit(line_atexit);
+        atexit_registered = true;
+    }
+
     if (!isatty(STDIN_FILENO)) {
         /* Not a tty: read from file / pipe. In this mode we don't want any
          * limit to the line size, so we call a function to handle that. */
```

至此，所有 `make valgrind` 會產生的記憶體問題都解決了。

### 使用 address sanitizer

重新編譯有開啟 address sanitizer 的版本。

```
make clean
make SANITIZER=1
```

之後執行 `make test` ，沒有出現任何記憶體相關錯誤訊息。至此檢查通過。

## Linux 核心 Linked List 和排序研究



## 貢獻記錄

### 修正 GitHub Action 問題

我檢視我的 repository 的 GitHub Action 有無正常運作時，發現 GitHub Action 在 `Run webfactory/ssh-agent@v0.7.0` 這個步驟時就失敗了。

於是我就看了一下原始的 `sysprog21/lab0-c` repository 。裡面的 GitHub Action 可以運作成功。發現原來 `Run webfactory/ssh-agent@v0.7.0` 是用來載入 ssh private key 讓原始 `sysprog21/lab0-c` repository 在沒有 `queue.c` 的解答情況下，能從另一個 private repository 拷貝 `queue.c` 來使用，使原始的 `sysprog21/lab0-c` repository 的 GitHub 在沒解答情況下也可以測試。

但是在 fork 之後的 repository ，不會有 ssh private key 。導致在 `Run webfactory/ssh-agent@v0.7.0` 步驟就失敗，以致後面的 `make check` 、 `make test` 等步驟都不會執行。

我看了 GitHub 的文件後，發現可以把 GitHub Action 某個步驟標示為即使失敗仍然能繼續執行。故我在 `.github/workflows/main.yml` 裡進行小修正，針對 `webfactory/ssh-agent@v0.7.0` 這個步驟加入 `continue-on-error: true` 來讓它失敗時，也可以往下執行其他 GitHub Action ，使得即便沒有 ssh private key ， GitHub Action 也能有作用。

我為此建了一個 [pull request](https://github.com/sysprog21/lab0-c/pull/112) 來提交我的改動，目前已經被 merge 了。


```diff
diff --git a/.github/workflows/main.yml b/.github/workflows/main.yml
index 56c7d9e..6ed0b93 100644
--- a/.github/workflows/main.yml
+++ b/.github/workflows/main.yml
@@ -8,6 +8,7 @@ jobs:
     steps:
     - uses: actions/checkout@v3.3.0
     - uses: webfactory/ssh-agent@v0.7.0
+      continue-on-error: true
       with:
           ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
     - name: install-dependencies
```