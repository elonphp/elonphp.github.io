---
title: "OCAdmin"
description: "我以 Laravel 開發的 OpenCart 管理系統 — 設計脈絡、實作心得與決策記錄"
---

**OCAdmin** 是我以 Laravel 為基礎開發的 OpenCart 後台管理系統。

這個區塊放的是**白話版的專案筆記**：

- 系統怎麼設計、為什麼這樣設計
- 實作上踩過的坑、做過的取捨
- 跟既有 OpenCart 生態整合的心得
- 各個模組的設計理念

如果你想看技術細節（API、設定欄位、CLI 指令），請到 GitHub repo 的 `docs/` 目錄；
這裡偏向**「為什麼」**而不是**「怎麼用」**。

---

## 系列文章

1. [介紹：為什麼選 OpenCart 後台作為設計藍本](/ocadmin/introduction/)
2. [整體架構：Portal、Core、Module 怎麼切，為什麼這樣切](/ocadmin/architecture/)
3. [多語機制：介面、網址、內容三層獨立設計](/ocadmin/multilingual/)
4. [命名規範：橫向單複數 + 縱向階層對齊](/ocadmin/naming/)
5. [權限機制：四段式命名、角色設計、Prefix 解耦](/ocadmin/permissions/)

---

每篇都盡量獨立可讀，不要求按順序看；第一次接觸這個系列建議從 #1 開始。
