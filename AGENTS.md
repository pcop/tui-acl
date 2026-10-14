# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概述

單一 bash 腳本 `tui-acl`：以 fzf 為介面的 Samba AD / Linux POSIX ACL 管理面板。沒有建置流程、沒有測試框架、沒有相依套件管理。使用者文件在 `README.md`，功能與安全性說明以該檔為準。

回覆與註解使用正體中文。

## 常用指令

```bash
bash -n tui-acl                                  # 語法檢查，每次修改後必跑
sudo ./tui-acl /path/to/target                   # 實際執行，必須是 root

# 不需 root 即可測試的內部子指令（在 root 檢查之前處理）
TUI_ACL_TARGET=/some/dir ./tui-acl --dump-menu
TUI_ACL_TARGET=/some/dir ./tui-acl --delete-entry "ITEM:a:user:alice"

# 單獨抽出預覽腳本來測試
sed -n "/^cat << 'PREVIEW_EOF'/,/^PREVIEW_EOF/p" tui-acl | sed '1d;$d' > /tmp/x/ph.sh
TUI_ACL_TARGET=/some/dir bash /tmp/x/ph.sh

# 測試 setfacl 行為時，建一個帶 mask 限制的目錄
mkdir t && setfacl -m u:nobody:rwx t && setfacl -m m::r-x t && getfacl -cp t
```

shellcheck 未安裝。本機 fzf 為 0.44.1，沒有 `--with-shell`，不要用 0.46 以後的 fzf 功能。

## 架構

### 單一腳本、三種執行模式

同一支腳本會以三種身份被執行，順序寫在檔案前段：

1. **`--dump-menu`**：由 fzf 的 `reload()` 呼叫，輸出主選單。
2. **`--delete-entry TAG`**：由 fzf 的 `execute()` 呼叫，刪除單一條目。有 TTY，失敗時印錯誤並 `read` 等待。
3. **主程式**：root 檢查、建立私有目錄、寫出預覽腳本、進入主迴圈。

前兩種在 root 檢查之前處理，所以可以非 root 測試。它們只依賴 `build_main_menu` 與環境變數 `TUI_ACL_TARGET`，不依賴快取目錄。

### 目標路徑如何傳遞

`TARGET` 只在主程式內使用。所有子程序（預覽腳本、`--dump-menu`、`--delete-entry`）一律讀環境變數 `TUI_ACL_TARGET`，主迴圈每輪開頭 `export` 一次。不要把 `$TARGET` 內插進 fzf 的 `--preview` 或 `--bind` 字串，那會在路徑含引號時壞掉。腳本自身路徑放進 bind 時用 `SELF_Q`（已 `printf %q`）。

預覽腳本 `write_preview_helper` 以 heredoc 寫到 `$CACHE_DIR/preview.sh`。它接受可選的位置參數（瀏覽模式用 fzf 的 `{1}` 傳路徑），沒有參數時退回 `TUI_ACL_TARGET`。

### fzf 行的 tag 協定

主選單每行是 `TAG<TAB>顯示文字`，fzf 用 `--delimiter='\t' --with-nth=2` 只顯示第二欄，程式邏輯只看第一欄。Tag 格式：

- `ACT:<NAME>` 功能項目
- `SEP` 分隔線
- `ITEM:a:<user|group>:<name>` 現行條目，`ITEM:d:...` 預設繼承條目

瀏覽模式則是 `完整路徑<TAB>顯示文字`。任何會進入 tag 欄的字串都不能含 tab。

### getfacl 輸出的解析陷阱

受 mask 限制的條目輸出為 `user:bob:rwx<TAB>#effective:r-x`。三個解析點都必須先剝除 `[[:space:]]+#.*$`，否則 tab 會破壞欄位：`build_main_menu`、`get_entities` 裡的 awk 輸入、propagate 的範本檔。用 `+` 不用 `*`，因為帳號名可能含 `#`。`\040` 要先轉回空白（AD 群組名常含空白，如 `domain users`）。

### 私有目錄與安全邊界

`setup_private_dir` 以 700 建立 `/var/cache/tui-acl` 與 `/var/backups/tui-acl`，並拒絕 symlink 或非本人擁有的目錄。這是刻意不用 `/tmp` 的原因：腳本以 root 執行，`/tmp` 下可預測的路徑會讓其他本機使用者替換 root 之後會讀取或執行的檔案。新增任何暫存檔都放 `$CACHE_DIR`，不要放 `/tmp`。

### 錯誤回報原則

腳本內沒有任何 `setfacl ... 2>/dev/null`，請維持。`setfacl -x` 對不存在的條目本來就是靜默 no-op（對檔案下 `d:` 也是），所以任何 stderr 都是真正的錯誤。模式是捕捉 `2>&1` 到變數，失敗時紅字顯示並 `read -r -p` 等待 Enter。propagate 則把 stderr 導到 `$CACHE_DIR/propagate-errors.log` 並計算行數，只有零才顯示 ✅。

## 已知陷阱

- **`GROUPS` 是 bash 保留變數**，賦值會被忽略且展開成執行者的主要 GID。選取結果用 `SEL_GROUPS` / `SEL_USERS`。
- `setfacl --mask -m m::---` 在沒有 mask 的目標上會憑空新增 mask；`d:m::` 在沒有 default ACL 的目錄上會建立整組 default 條目。RECALC_MASK 已有守衛，修改時保留。
- `find -printf '.' | wc -c` 是刻意的，用來讓計數對含換行的路徑免疫。
- 腳本檔曾是 root:root，若無法直接 Edit，改在 scratchpad 修改後以 `cp` + `mv` 替換（目錄可寫），擁有者會變成目前使用者。

## 其他

Git repo 已初始化但尚無任何提交。使用者偏好先在副本上修改並以 `diff` 檢視，確認後再安裝。
