# xv6 Lab Notes

這個 repo 用來整理我實作 xv6 labs 的中文筆記、除錯紀錄與測試結果。
公開內容以設計思路、debug 過程與少量關鍵 diff 說明為主，不包含完整作業原始碼。

## 內容

- [`pgtbl` lab 筆記](./docs/pgtbl.md)

## 這份 repo 想呈現的重點

- 我如何理解作業需求，而不只是把功能做出來
- 我如何定位 kernel panic、記憶體重疊與 page table 問題
- 我如何用 `QEMU + GDB` 追出 root cause
- 我如何驗證修改是否真的通過 grader

## 技術關鍵字

- C
- Operating Systems
- xv6
- RISC-V
- Sv39 page table
- QEMU
- GDB
- kernel / user address space
- memory mapping

## 這次作業摘要

本次紀錄的是 `pgtbl` lab，重點包含兩部分：

1. 為每個 process 建立自己的 kernel page table
2. 簡化 `copyin` / `copyinstr`，讓 kernel 能直接透過 process 的 kernel page table 存取 user pointer

## 最終測試結果

```text
make grade
Score: 66/66
```

## Repo 結構

```text
xv6-lab-notes/
├── README.md
├── docs/
│   └── pgtbl.md
└── assets/
```

## 說明

基於課程與學術規範，本 repo 不公開完整 lab 實作程式碼。
內容保留高層設計、關鍵改動方向、debug 紀錄與測試結果，方便日後回顧，也方便作為個人學習歷程展示。
