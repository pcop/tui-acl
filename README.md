# tui-acl

Samba AD / Linux POSIX ACL 互動式全螢幕管理面板。以 fzf 為介面，讓管理者在終端機內瀏覽目錄、即時檢視 ACL，並以精靈方式新增、修改、刪除與批次套用權限。

## 需求

| 項目 | 說明 |
|---|---|
| 執行身份 | 必須以 root 執行，非 root 會直接結束並提示改用 `sudo` |
| bash | 4.0 以上 |
| fzf | 0.44 以上（使用 `--bind`、`execute`、`reload`、`--expect`） |
| acl 套件 | 提供 `getfacl` / `setfacl` |
| GNU coreutils / findutils | 使用 `find -printf`、`readlink -f`、`realpath`、`stat -c` |
| winbind（選用） | 偵測到 `wbinfo` 且 winbind 在線時，自動列出 AD 使用者與群組 |

目標路徑所在的檔案系統必須支援 POSIX ACL（ext4、xfs 等；NFS 或某些掛載選項可能不支援）。

## 安裝

```bash
sudo install -m 755 -o root -g root tui-acl /usr/local/sbin/tui-acl
```

## 使用方式

```bash
sudo tui-acl /srv/share/dept      # 指定目標目錄或檔案
sudo tui-acl                      # 不給參數時以目前目錄為目標
```

### 主選單

| 項目 | 說明 |
|---|---|
| 瀏覽 / 切換目標路徑 | 在目錄樹中移動，右側即時顯示游標所在項目的 ACL。選「鎖定此目錄本身」或選到檔案即切換目標 |
| 新增 / 修改 使用者 | 三步精靈：搜尋並以 Tab 勾選多位使用者、選擇權限、選擇套用範圍 |
| 新增 / 修改 群組 | 同上，對象為群組 |
| 將目前資料夾權限套用至下層 | 危險操作，見下節。只在目標為目錄時出現 |
| 使用文字編輯器批次編輯 | 以 `$EDITOR`（預設 vim）編輯 getfacl 輸出後整份套用 |
| 重新計算有效遮罩 | 以所有群組與具名條目的聯集重算 mask，修復 `#effective` 降權。沒有 mask 或 default ACL 的目標不會被動到 |
| 清除所有擴充 ACL | `setfacl -b`，可選僅此路徑或遞迴整棵樹 |

選單下方列出目標現有的每一條使用者與群組 ACL，分為「現行存取」與「預設繼承」兩類。受 mask 限制的條目會額外標示有效權限。

### 快捷鍵

| 按鍵 | 動作 |
|---|---|
| Enter | 執行選單項目，或對現有條目開啟修改 / 刪除子選單 |
| Ctrl-D | 直接刪除游標所在的 ACL 條目，失敗時顯示錯誤並等待 Enter |
| Ctrl-R | 強制重新整理 AD 使用者與群組快取（預設 10 分鐘） |
| Esc | 返回上一層或離開 |

### 權限選項

| 代碼 | 意義 |
|---|---|
| `rwx` | 完整控制 |
| `r-X` | 讀取，只對目錄給予進入權限（推薦） |
| `rwX` | 讀寫，只對目錄給予進入權限 |
| `r-x` | 讀取並強制給予執行權限 |
| `---` | 掛載空權限以阻擋特定對象 |

### 套用範圍（目標為目錄時）

| 編號 | 行為 |
|---|---|
| 1 | 此目錄本身加上 default 繼承規則，新建項目自動套用 |
| 2 | 遞迴所有現有子項目，並對每個子目錄設定 default 繼承 |
| 3 | 只修改此目錄本身 |
| 4 | 只遞迴現有子項目，不設定 default 繼承 |

## 將目前資料夾權限套用至下層

把目標目錄的 ACL 當成範本，強制覆寫整棵子樹，下層原有的個別 ACL 會被清除。有三道確認：選擇範圍、選擇 YES、在終端機輸入大寫 `YES`。

| 模式 | 行為 |
|---|---|
| DIRS | 只覆寫子目錄，不動檔案 |
| ALL_SAFE | 覆寫子目錄與檔案，檔案的 x 權限一律移除（推薦） |
| ALL_RAW | 覆寫子目錄與檔案，檔案保留與目錄相同的 x 權限 |

執行前會把整棵子樹的現有 ACL 備份到 `/var/backups/tui-acl/acl-backup-<日期時間>.facl`。備份失敗會中止，不做任何變更。完成後會逐階段統計 setfacl 回報的錯誤筆數，只有零錯誤才顯示成功；有錯誤時列出前 10 筆與完整記錄檔位置。

還原方式：

```bash
setfacl --restore=/var/backups/tui-acl/acl-backup-20261014-093000.facl
```

## 檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/var/cache/tui-acl/` | 預覽腳本、AD 帳號快取、propagate 錯誤記錄、編輯失敗時保留的暫存檔。啟動時以 700 建立，若不是 root 擁有的真實目錄會拒絕執行 |
| `/var/backups/tui-acl/` | propagate 的 ACL 備份，權限 600 |

`/var/cache/tui-acl/edit.*` 只會在文字編輯器模式套用失敗時留下，內含你的編輯內容，可修正後以提示的指令重試，或直接刪除。

## 內部子指令

以下由 fzf 子程序呼叫，目標路徑透過環境變數 `TUI_ACL_TARGET` 傳遞，一般不需手動使用：

```bash
TUI_ACL_TARGET=/srv/share tui-acl --dump-menu          # 輸出主選單內容
TUI_ACL_TARGET=/srv/share tui-acl --delete-entry TAG   # 刪除單一條目，TAG 形如 ITEM:a:user:alice
```

## 安全性說明

- 腳本內不呼叫 sudo，所有 getfacl 與 setfacl 都以執行者身份進行，因此必須是 root。
- 快取與備份不放在 `/tmp`，避免其他本機使用者預先佔用路徑後讓 root 讀取或執行被竄改的檔案。
- 所有 setfacl 失敗都會顯示在畫面上，不再靜默吞掉。
- 清除所有擴充 ACL 的遞迴模式只有一層確認且不備份，請依賴外部快照機制。

## 已知限制

- 路徑含 tab 或換行字元會破壞瀏覽模式的欄位分隔。
- 帳號或群組名稱含 ` [` 會在勾選後被截斷。
- `winbind separator` 設定會影響 AD 帳號在清單中的顯示格式，需與 getfacl 的輸出一致。
