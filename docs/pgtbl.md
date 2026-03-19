# xv6 `pgtbl` Lab 筆記

## 作業目標

這個 lab 分成兩部分：

1. `A kernel page table per process`
2. `Simplify copyin/copyinstr`

第一部分的目標，是把 xv6 原本全系統共用的一張 `kernel_pagetable`，改成每個 process 都有自己的一張 kernel page table。

第二部分的目標，是讓 `copyin()` / `copyinstr()` 不必每次都手動 walk user page table 找 physical address，而是可以直接透過該 process 的 kernel page table 存取 user virtual address。

---

## 我一開始的理解

一開始我對這題最容易搞混的地方是：

- user page table 是給 user mode 用
- kernel page table 是 process 進入 kernel 後使用

更精確地說，每個 process 會同時擁有兩張表：

1. user page table
2. kernel page table

第二部分不是把 user page table 取代掉，而是讓「該 process 的 kernel page table 也能看見它自己的 user mappings」。

---

## Part 1：Per-process kernel page table

### 核心改動

- 在 `struct proc` 新增 `kernel_pagetable`
- `allocproc()` 建立新的 kernel page table
- 每個 process 的 kernel stack 改成 map 到自己的 kernel page table
- `scheduler()` 在 context switch 前後切換 `satp`
- `freeproc()` 回收該 process 的 kernel page table

### 我理解的重點

原本 xv6 在 kernel mode 執行時，所有 process 都共用同一張 kernel page table。
這題要求把它改成「每個 process 都有一份自己的 kernel page table」，而且在 scheduler 選到某個 process 時，CPU 要切換到那個 process 自己的 kernel page table。

這一部分的難點不在新增欄位，而是在 lifecycle：

- 什麼時候建立
- 什麼時候切換
- 什麼時候回收

### 驗證方式

這一部分的驗證重點是：

- `usertests` 要能正常通過

---

## Part 2：Simplify copyin/copyinstr

### 核心改動

- `copyin()` 改成呼叫 `copyin_new()`
- `copyinstr()` 改成呼叫 `copyinstr_new()`
- process 的 kernel page table 需要同步 user mappings

### 哪些時機需要同步 user mappings

只要 user address space 發生變化，kernel page table 的對應部分也要一起更新。

這次我實際處理到的同步點有：

- `userinit()`
- `fork()`
- `exec()`
- `growproc()` / `sbrk()`

### 權限重點

把 user page 映射進 kernel page table 時，不能保留 `PTE_U`。

原因是：

- `PTE_U` 是給 user mode 存取的標記
- kernel mode 存取 user address 時，這個 mapping 必須存在於 kernel page table 中
- 但如果保留 `PTE_U`，權限語意會不符合這題要做的事

所以同步 user mapping 到 kernel page table 時，要把 `PTE_U` 清掉。

### user memory 上限

這題還有一個很重要的限制：

- user VA 不能和 kernel 已經保留的 VA 範圍重疊

在 xv6 這個 lab 的設定下，user process 的大小要限制在 `PLIC` 之前，也就是小於 `0x0c000000`。

---

## 我踩到的幾個問題

### 1. `copyin_new()` / `copyinstr_new()` 明明有實作，卻編譯失敗

我一開始把 `copyin()` / `copyinstr()` 改成直接呼叫 `copyin_new()` / `copyinstr_new()` 後，`make grade` 先卡在編譯：

```text
implicit declaration of function 'copyin_new'
implicit declaration of function 'copyinstr_new'
```

原因不是函式不存在，而是：

- `kernel/vmcopyin.c` 有定義
- 但 `kernel/vm.c` 看不到它們的 prototype

解法是：

- 在 `kernel/defs.h` 加上這兩個函式的宣告

這提醒我一件事：

- 「已經有實作」不等於「其他檔案可見」

### 2. `panic: remap`

這是整個 lab 最重要的 bug。

一開始 `make grade` 會在 `usertests` 很早的地方炸掉：

```text
panic: remap
```

我原本先懷疑是：

- `growproc()` 同步 user mapping 到 kernel page table 時，起始位址沒對齊

這個猜測有一部分是對的，但不是最後的 root cause。

### 3. `bigdir` 失敗其實不一定是 page table bug

我在第一部分也碰過：

```text
bigdir: bigdir link(bd, x00) failed
```

後來檢查發現，那次不是 page table 邏輯本身錯，而是 `fs.img` 已經被之前的測試跑髒了，裡面殘留 `x00`、`x01` 等檔名。

把 `fs.img` 清掉後重跑，`bigdir` 就正常通過。

這件事提醒我：

- 測試失敗時，要先分辨是「程式邏輯錯誤」還是「測試環境殘留狀態」

---

## 我怎麼用 GDB 把 `remap` 找出來

這次我用的是非常典型的 kernel debug 流程：

1. 用 `make qemu-gdb` 啟動 xv6
2. 用 `gdb-multiarch -q kernel/kernel` 載入 symbols
3. 用 `target remote :26000` 接到 QEMU
4. 在 `panic` 下 breakpoint
5. 執行 `usertests`
6. 程式停在 `panic` 時，用 `bt` 看 backtrace
7. 逐層切 frame，印出參數與位址

### 實際使用的指令

```gdb
target remote :26000
b panic
c
bt
frame 1
frame 2
frame 3
list
p/x some_var
```

### 抓到的關鍵呼叫鏈

backtrace 顯示：

```text
panic("remap")
-> mappages()
-> u2kvmcopy()
-> growproc()
-> sys_sbrk()
```

這個結果很重要，因為它直接排除了：

- `exec()` 不是第一嫌疑
- `fork()` 不是第一嫌疑

而是明確指出：

- 問題是 `growproc()` 這條路在做 `u2kvmcopy()` 時發生的

### 關鍵位址

我接著印出 `mappages()` 當下的 `va`，看到的是：

```text
0x02000000
```

對照 `memlayout.h` 後就知道：

- 這個位址是 `CLINT`

也就是說，當 process 用 `sbrk()` 長大到那個範圍時，user mapping 和 kernel 原本保留的 `CLINT` mapping 發生碰撞，所以 `mappages()` 才會 `panic("remap")`。

### 最後的 root cause

根因不是 `copyin_new()` 自己壞掉，而是：

- 我把 user mapping 加進 process 的 kernel page table
- 同時又保留了 `CLINT` mapping
- 但這題的設計前提是 user memory 上限要到 `PLIC`
- 因此 kernel page table 的低位址保留範圍不能先被 `CLINT` 佔住

### 解法

- 把 `CLINT` mapping 從 `kvminit()` 拿掉
- 也從 per-process 的 `proc_kpagetable()` 拿掉
- 以 `PLIC` 作為 user process size 上限

這樣 user VA 和 kernel reserved VA 範圍才一致。

---

## 少量關鍵修改方向

以下只保留概念級的 diff，不公開完整作業解答：

```diff
- one global kernel page table for all processes
+ one kernel page table per process
```

```diff
- copyin() walks the user page table in software
+ copyin() delegates to copyin_new()
```

```diff
- kernel page table only maps kernel addresses
+ process kernel page table also mirrors that process's user mappings
```

```diff
- keep CLINT mapped in kernel page tables
+ remove CLINT mapping to avoid overlap before PLIC
```

---

## 測試結果

我用以下方式驗證：

```bash
make
make grade
```

最終 grader 結果：

```text
Score: 66/66
```

---

## 這次最大的收穫

這次 lab 幫我真的把幾件事串起來：

- user page table 與 kernel page table 的角色分工
- 為什麼 `copyin_new()` 的前提是 kernel page table 必須能看見 user mapping
- 記憶體衝突不一定是 code typo，常常是 address range 設計不一致
- GDB 真正有用的不是指令多炫，而是：
  - 先停在 crash 點
  - 再看 backtrace
  - 再驗證位址與邊界條件

如果之後要再做類似 kernel / memory management 題目，我會優先做這三件事：

1. 先確認重現條件
2. 再抓 crash 現場
3. 最後才回頭改 code
