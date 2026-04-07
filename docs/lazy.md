# xv6 `lazy` Lab 筆記

## 作業目標

這個 lab 的核心是把 xv6 原本「`sbrk()` 立刻配置實體記憶體」的行為，改成 lazy allocation。

也就是說：

1. `sbrk(n)` 先只增加 process 的位址空間大小
2. 不立即配置 physical page
3. 等 user program 第一次真正存取該位址時，再由 page fault handler 補配置頁面

後半段還要把這個設計補完整，讓 `lazytests` 與 `usertests` 都能通過。

---

## 我一開始的理解

這題最容易誤會的地方是：

- `sbrk()` 不再等於「直接拿到可以用的實體記憶體」
- `sbrk()` 只代表「這段 virtual address 現在合法了」

換句話說，process 的 `sz` 變大之後，位址空間是有了，但其中很多 page 其實還沒有對應到 physical memory。

真正的配置時機變成：

- user code 存取該 page
- CPU 觸發 page fault
- kernel 在 trap handler 裡補上 mapping

---

## 核心修改

### 1. `sys_sbrk()` 不再立即配置頁面

在 `sys_sbrk()` 中：

- 當 `n > 0` 時，只增加 `p->sz`
- 當 `n < 0` 時，改用 `uvmdealloc()` 正確回收頁面

這一部分的關鍵，不是把 `growproc()` 拿掉而已，而是要維持 `p->sz` 的一致性。

我這次實際踩到的 bug 是：

- `addr` 如果還用 `int`，在 RV64 上會有位址截斷問題
- `sbrk(-n)` 不能只做 `p->sz += n`，而要用 `uvmdealloc()` 的回傳值更新 `p->sz`

---

### 2. 在 `usertrap()` 中處理 page fault

當 `r_scause()` 是：

- `13`：load page fault
- `15`：store page fault

kernel 會：

1. 讀出 faulting address：`r_stval()`
2. 用 `PGROUNDDOWN()` 對齊到 page boundary
3. 呼叫 `kalloc()` 配一頁
4. `memset()` 清零
5. `mappages()` 建立 user mapping

如果位址不合法，或 `kalloc()` 失敗，就 kill 該 process。

---

### 3. `walkaddr()` 也要支援 lazy allocation

只改 `usertrap()` 還不夠。

原因是很多情況不是 user code 直接觸發 page fault，而是 kernel 在 system call 中透過：

- `copyin()`
- `copyout()`
- `copyinstr()`

去存取 user buffer。

所以我也修改了 `walkaddr()`：

- 如果 page table entry 不存在，但位址仍在合法範圍內
- 就在 `walkaddr()` 裡補配置該頁面

這樣像 `read()`、`write()`、`pipe()`、`exec()` 之類使用 user pointer 的系統呼叫才會正常。

---

### 4. `uvmunmap()` 不能再假設所有頁面都已經存在

在 lazy allocation 設計下，`p->sz` 可能已經很大，但其中很多頁面其實從未被實際配置。

所以在 `uvmunmap()` 中：

- 如果 `walk()` 找不到 page table entry，不能 panic
- 如果 PTE 無效，也不能 panic

這些情況都應該直接略過。

否則：

- `exit()`
- `exec()`
- `sbrk(-n)`

都可能因為碰到「邏輯上存在、實際上尚未配置」的頁面而炸掉。

---

### 5. `fork()` 要正確處理尚未配置的 lazy page

`uvmcopy()` 原本假設 parent address space 內每一頁都真的存在。

但在這題中不是這樣：

- parent 的某些頁面只是被 `sbrk()` 保留
- 還沒有真正觸發 page fault

因此在 `uvmcopy()` 中：

- 遇到不存在的 PTE 或 invalid page 時要跳過
- 只複製真正有實體頁面的部分

這樣 child 才能保留正確的 lazy allocation 狀態。

---

## 合法位址範圍的判斷

這題還有一個容易忽略的點：

- 不是所有 page fault 都應該自動配置頁面

我最後的判斷條件是：

- `va < p->sz`
- `va >= PGROUNDDOWN(p->trapframe->sp)`

意思是：

- faulting address 不能超過目前 process 已分配的 user address space
- 也不能落到 stack 下方那個無效 guard page

這樣才能同時通過：

- 合法 heap lazy allocation
- stack 下方非法頁面的保護測試

---

## 我踩到的幾個問題

### 1. `sys_sbrk()` 看起來改很少，但其實很容易留下 hidden bug

一開始我把 `growproc()` 拿掉之後，直覺上覺得只要：

```c
p->sz += n;
```

就可以了。

但後來發現這樣在 `n < 0` 時不夠安全，因為：

- 需要真的解除映射
- `p->sz` 必須和 `uvmdealloc()` 的結果一致

這也是 `sbrkbugs` 測試會抓的問題。

---

### 2. 只改 page fault handler，不改 `walkaddr()` 會卡在 syscall 測試

一開始最容易以為：

- user page fault 都在 `usertrap()` 裡處理掉就夠了

但 `sbrkarg`、`copyin`、`copyout`、`copyinstr` 相關測試會提醒你：

- kernel 自己也會去碰 user memory
- 這時候不會走 user trap path

所以 `walkaddr()` 也必須具備 lazy allocation 行為。

---

### 3. 測試全掛不一定是 lazy allocation 邏輯本身錯

我這次還額外碰到一個和環境有關的問題：

- 新版 QEMU / toolchain 下，kernel 一開始會卡在 boot 過程
- 表面看起來像是 `lazytests`、`usertests` 全部失敗
- 但實際上 xv6 根本還沒正常進到 shell

我最後是用：

- `QEMU` log
- `kernel.asm`
- `gdb-multiarch`

把問題縮小到 boot path，才確認有額外的啟動相容性要處理。

這部分不是 lab 的核心邏輯，但如果環境較新，會直接影響 grader 是否能跑起來。

---

## 我怎麼定位問題

這次主要是用兩條線一起查：

### 1. 看 grader 掛在哪一類測試

我先跑：

```bash
make grade
```

觀察是：

- 一開始 `lazytests` 沒過
- 後來連 `usertests` 也全掛

這代表不能只看單一函式，而要先分清楚：

- 是 lazy allocation 邏輯錯
- 還是 kernel 連基本執行都還不穩定

### 2. 用 QEMU/GDB 檢查 boot 與 page fault 流程

我這次有實際用到：

```bash
make qemu-gdb
gdb-multiarch kernel/kernel
target remote :26000
```

然後配合：

- 看 `_entry` / `start()` / `main()`
- 看 page fault 時的 `scause`、`stval`
- 看 `kernel.asm` 對照 fault 位址

這樣可以把「環境問題」和「lab 實作問題」拆開來看。

---

## 少量關鍵修改方向

以下只保留概念級 diff，不公開完整作業解答：

```diff
- sbrk() grows memory by calling growproc()
+ sbrk() only updates process size, no immediate allocation
```

```diff
- unexpected user page fault => kill / panic path
+ legal user page fault => allocate page lazily and map it
```

```diff
- walkaddr() returns 0 when the page is missing
+ walkaddr() may allocate the missing lazy page for valid user addresses
```

```diff
- uvmunmap() / uvmcopy() assume every page below sz is already mapped
+ uvmunmap() / uvmcopy() must tolerate unmapped lazy pages
```

---

## 最終測試結果

```text
make grade
Score: 119/119
```

代表：

- `lazytests` 通過
- `usertests` 通過
- lazy allocation 的主要邏輯與邊界情況都正確處理完成

---

## 這次我學到的事

這題表面上像是在做一個小優化，但實際上它牽動的是整個「虛擬位址合法、但實體頁面尚未存在」的模型。

也就是說，修改的不只是：

- `sbrk()`

而是整個 kernel 對 user address space 的假設。

我覺得這題最重要的收穫有三個：

1. `virtual memory` 的合法性和 `physical page` 的存在是兩件事
2. kernel 中所有會碰到 user memory 的路徑，都要重新檢查這個假設
3. 測試全掛時，不能先入為主地認定一定是 lab 核心邏輯錯，也要懷疑 boot / toolchain / QEMU 相容性
