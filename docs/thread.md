# xv6 `thread` Lab 筆記

## 作業目標

這個 lab 分成三部分：

1. `Uthread: switching between threads`
2. `Using threads`
3. `Barrier`

這次的主題很集中在 multithreading，但三題其實分別對應到三種不同層次：

- user-level thread context switch
- shared data structure 的同步與效能
- thread coordination / barrier synchronization

---

## 我一開始的理解

這次我覺得最重要的，不是單純「把 thread 跑起來」而已，而是把下面三件事分清楚：

- `uthread` 的重點是 context switch，不是 kernel thread
- `ph` 的重點不只是 correctness，還有 lock granularity
- `barrier` 的重點不只是叫醒別人，而是正確處理一輪又一輪的同步

也就是說，這個 lab 雖然都叫 multithreading，但三題其實在練：

1. CPU register state 怎麼保存
2. shared memory race 怎麼避免
3. 多執行緒怎麼在 round-based 協作裡保持一致

---

## Part 1：Uthread

### 核心改動

- 在 `user/uthread.c` 新增 `struct context`
- context 內保存 `ra`、`sp` 與 `s0 ~ s11`
- `thread_create()` 初始化新 thread 的起始 `ra` 與 `sp`
- `thread_schedule()` 在 old thread / new thread 之間呼叫 `thread_switch()`
- `user/uthread_switch.S` 負責保存舊 thread 的 callee-saved registers，並恢復新 thread 的 registers

### 我理解的重點

這題的本質是：

- thread 被切走時，要把「它停在哪裡」完整記下來
- 下次切回來時，要從那個位置繼續跑

這個「狀態」最核心的就是 register context。

我最後的做法是把每個 thread 的執行狀態放進：

```c
struct context {
  uint64 ra;
  uint64 sp;
  uint64 s0;
  ...
  uint64 s11;
};
```

然後把它掛在 `struct thread` 裡，讓 scheduler 切換時直接對這塊記憶體存取。

### 為什麼只需要保存 callee-saved registers

這題提示說 `thread_switch` 只要存 callee-saved registers。

我的理解是：

- caller-saved registers 本來就不保證跨函式呼叫還存在
- `thread_switch()` 可以把自己想成一個普通函式呼叫邊界
- 因此真正需要保留的是 ABI 規定必須被 callee 保留的那些 register

也就是：

- `ra`
- `sp`
- `s0 ~ s11`

### 第一次執行新 thread 是怎麼開始的

這題最關鍵的地方，是讓一個「從來沒跑過」的 thread 也能被 scheduler 第一次切進去。

我最後是這樣做：

- `t->context.ra = func`
- `t->context.sp = t->stack + STACK_SIZE`

這樣當 scheduler 第一次 restore 這個 thread 的 context 時：

- `sp` 已經指到它自己的 stack 頂端
- `ra` 已經是該 thread 要執行的函式位址

接著 `thread_switch()` 最後一個 `ret`，就會直接跳進 `func()` 開始執行。

### 實作上的關鍵

`thread_schedule()` 的關鍵呼叫最後變成：

```c
thread_switch((uint64)&t->context, (uint64)&current_thread->context);
```

也就是：

- 第一個參數是舊 thread 的 context，拿來存現在 CPU 上的狀態
- 第二個參數是新 thread 的 context，拿來把 CPU 恢復成它上次離開時的樣子

而 `thread_switch.S` 內就是成對地：

- `sd` 存舊 thread registers
- `ld` 載入新 thread registers
- `ret`

### 我覺得這題最容易卡住的點

- 如果 `sp` 沒設到 stack 頂端，新 thread 一開始就會在錯的 stack 上執行
- 如果 `ra` 沒設成 `func`，第一次 `ret` 根本不知道要跳去哪裡
- 如果 scheduler 沒先把 `current_thread` 換成 `next_thread`，thread bookkeeping 會亂掉

---

## Part 2：Using Threads with Hash Table

### 為什麼兩個 threads 會導致 key 消失

這題我覺得最重要的是先把 race condition 說清楚。

假設兩個 threads 同時對同一個 bucket 做 `put()`，可能發生下面這串事件：

1. 兩個 thread 同時算出同一個 bucket `i`
2. 兩邊都掃描 linked list，發現 key 還不存在
3. 兩邊都準備把新節點插到 `table[i]` 的表頭
4. thread A 先把 `table[i]` 改成自己的新節點
5. thread B 接著又把 `table[i]` 改成自己的新節點
6. thread A 那個剛插進去的節點就被蓋掉了

結果就是：

- 其中一個 key 明明有 insert 過
- 最後卻從 bucket 鏈結串列裡消失

這也是為什麼 `ph 1` 沒事，但 `ph 2` 會出現 missing keys：

- 單執行緒沒有 concurrent update
- 雙執行緒同時改同一個 bucket head 時就會發生 lost update

### 核心改動

我最後沒有用「整張 hash table 一把大鎖」，而是改成：

```c
pthread_mutex_t locks[NBUCKET];
```

也就是每個 bucket 一把 lock。

對應修改是：

- `main()` 中先把每把 bucket lock 都 `pthread_mutex_init()`
- `put()` 裡根據 `key % NBUCKET` 取得對應 bucket lock
- 對該 bucket 的 search / insert / update 全部包在 lock 內

這樣的好處是：

- 同一個 bucket 的更新仍然被序列化，能保證 correctness
- 不同 bucket 的 `put()` 可以平行進行，保留 parallel speedup

### 關於 `get()` 我最後的取捨

題目一開始提到可以在 `put()` 和 `get()` 加鎖，但這份 benchmark 的流程其實是：

1. 所有 `put()` threads 完成
2. `pthread_join()` 全部結束
3. 之後才開始 `get()` benchmark

也就是 `get()` 發生時，hash table 已經不再被 concurrent modification。

所以在這個 lab 的測試條件下，我最後只對 `put()` 的 bucket update 上鎖，就能同時通過：

- `ph_safe`
- `ph_fast`

不過如果把這份程式當成一般化的 concurrent hash table，那 `get()` 也應該納入同步設計，否則和 concurrent `put()` 混用時仍然不安全。

### 本機實測結果

我這次在目前這台機器上跑到的其中一組數字是：

```text
$ ./ph 1
100000 puts, 3.418 seconds, 29257 puts/second
0: 0 keys missing
100000 gets, 3.410 seconds, 29327 gets/second

$ ./ph 2
100000 puts, 2.492 seconds, 40131 puts/second
1: 0 keys missing
0: 0 keys missing
200000 gets, 3.578 seconds, 55899 gets/second
```

這組結果代表：

- correctness 已經修好，missing keys 是 `0`
- 兩個 threads 的 puts/second 大約是單執行緒的 `1.37x`
- 已經超過 `ph_fast` 要求的 `1.25x`

### 我理解的重點

這題最值得記的不是「要加 lock」而已，而是：

- 鎖太大，correct 但不快
- 鎖太小，快但會錯

per-bucket lock 剛好就是這題在 correctness 跟 speedup 之間的平衡點。

---

## Part 3：Barrier

### 這題真正要處理的是什麼

barrier 的目標是：

- 每個 thread 在某一輪到達 barrier 後，都必須等到所有其他 thread 也到達
- 等最後一個 thread 到了，這一輪才能一起放行

但這題麻煩的地方不只是一輪而已，而是：

- barrier 會在 loop 中重複使用很多次
- thread A 可能剛離開上一輪 barrier，就很快又跑回下一輪

所以不能只用單純的計數器，還要有 round 概念。

### 核心改動

我最後的 barrier 邏輯是：

1. 先拿 `barrier_mutex`
2. 記住自己進來時看到的 `my_round = bstate.round`
3. 把 `bstate.nthread++`
4. 如果自己是最後一個到達 barrier 的 thread：
   - 把 `bstate.nthread` 重設成 `0`
   - `bstate.round++`
   - `pthread_cond_broadcast()` 把其他人全部叫醒
5. 如果自己不是最後一個：
   - 用 `while (my_round == bstate.round)` 持續等待
6. 最後 unlock

### 為什麼一定要有 `round`

這題最容易忽略的，是 `nthread` 這個欄位會被下一輪重複使用。

如果沒有 `round`：

- 你很難區分「我現在是在等上一輪結束」還是「下一輪又有人進來了」
- thread 跑得快時，可能會提前把下一輪的 `nthread` 加進去
- 前一輪和下一輪的狀態就會混在一起

所以 `my_round` 的作用其實是：

- 每個 thread 都只等自己進來時所屬的那一輪結束
- 被叫醒後再檢查一次 round 是否真的變了

這也是用 `while` 而不是 `if` 的原因。

### 本機實測結果

我這次直接測了：

```text
$ ./barrier 1
OK; passed

$ ./barrier 2
OK; passed

$ ./barrier 4
OK; passed
```

代表 barrier 在不同 thread 數量下都能維持 round 同步，不再觸發原本的 assertion。

---

## 我踩到的幾個重點

### 1. `uthread` 比較像在做 ABI / context restore，而不是一般 scheduler 題

一開始如果只從「找下一個 runnable thread」的角度看，很容易低估這題。

真正的關鍵其實是：

- restore 完 register 後，CPU 要能自然回到那個 thread 原本的控制流

所以 `ra`、`sp` 的初始化和 `ret` 的落點，比 scheduler loop 本身還重要。

### 2. `ph` 的難點不是加鎖，而是 deciding lock scope

如果只想先讓 correctness 過，整張表一把大鎖很直覺。

但 grader 還會看 speedup，所以最後一定得思考：

- 哪些操作其實彼此互不干涉
- 哪些共享資料真的需要同一把鎖保護

### 3. `barrier` 最容易錯的是 round reuse

`pthread_cond_wait()` / `pthread_cond_broadcast()` 的 API 本身不難，
難的是：

- 怎麼讓同一個 barrier 被連續重複使用
- 又不讓上一輪和下一輪互相污染

我覺得這題最值得記下來的，就是「condition variable 幾乎總要和某個狀態條件一起使用」，這裡那個狀態條件就是 `round`。

---

## 我怎麼驗證

我這次實際跑的驗證包括：

```bash
make ph
./ph 1
./ph 2

make barrier
./barrier 1
./barrier 2
./barrier 4

make grade
```

在目前這份 checkout 中，`make grade` 的結果是：

```text
uthread: OK
ph_safe: OK
ph_fast: OK
barrier: OK
answers-thread.txt: FAIL
time: FAIL
Score: 54/60
```

也就是說：

- 核心實作測試都已經通過
- 目前缺的是 handin 需要的 `answers-thread.txt` 與 `time.txt`

---

## 少量關鍵修改方向

以下只保留概念級 diff，不公開完整作業解答：

```diff
- a thread has only stack/state
+ a thread also stores a saved register context
```

```diff
- the scheduler jumps to a fresh thread magically
+ thread_create() prepares ra/sp so first restore + ret enters func()
```

```diff
- concurrent puts race on a shared bucket head
+ each bucket has its own mutex, so independent buckets can proceed in parallel
```

```diff
- barrier waits only on a thread counter
+ barrier wait is guarded by both count and round number
```
