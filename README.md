# elonphp.github.io

我的個人網站與部落格，網址 https://elonphp.github.io/

## 本機開發

```powershell
# 啟動本機預覽 (http://localhost:1313)
hugo server

# 建立新文章 (Page Bundle 結構)
mkdir content\posts\2026-XX-XX-文章-slug
# 然後在裡面建 index.md 和圖片檔
```

## 部署

push 到 `main` 後，GitHub Actions 自動 build 並部署。
設定檔在 `.github/workflows/hugo.yml`。

## 主題

使用 [PaperMod](https://github.com/adityatelange/hugo-PaperMod)，以 git submodule 形式引入：

```powershell
# clone 時加 --recurse-submodules
git clone --recurse-submodules <repo-url>

# 已 clone 後初始化 submodule
git submodule update --init --recursive

# 更新主題
git submodule update --remote themes/PaperMod
```
