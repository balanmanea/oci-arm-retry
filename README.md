# OCI（Oracle Cloud Infrastructure）免費 VM 搶機腳本

## 兩種腳本

| 腳本 | 目標機型 | 規格 | 架構 |
|------|---------|------|------|
| `oci_retry.py` | VM.Standard.A1.Flex | 4 OCPU / 24 GB RAM / 200 GB 磁碟 | ARM (Ampere) |
| `oci_retry_micro.py` | VM.Standard.E2.1.Micro | 1 OCPU / 1 GB RAM / 50 GB 磁碟 | x86 |

兩種都是 Oracle Always Free 永久免費方案。每帳號最多 4 OCPU ARM（可拆成多台）+ 2 台 Micro。

---

## 腳本設定

開啟腳本，修改最上方的兩個變數：

```python
COMPARTMENT_ID = "ocid1.tenancy.oc1..你的OCID"
SSH_PUBLIC_KEY = "ssh-rsa AAAA...你的公開金鑰內容"
RETRY_INTERVAL = 90  # 重試間隔（秒）
```

**取得 Compartment ID**：Oracle Cloud 主控台 → 右上角頭像 → **我的設定檔** → 複製**租用戶 OCID**

**取得 SSH 公開金鑰**：在本機執行 `ssh-keygen`，複製 `.pub` 檔的完整內容（`ssh-rsa AAAA...` 開頭）

---

## 本機執行

```bash
# 安裝套件（只需一次）
pip install oci

# Windows 切換到資料夾
cd /d D:\Projects\oci_arm_retry

# 執行 ARM 版
python oci_retry.py

# 執行 Micro 版
python oci_retry_micro.py
```

本機執行需要 OCI 設定檔：`C:\Users\你的帳號\.oci\config`

```
[DEFAULT]
user=ocid1.user.oc1..xxxxxx
fingerprint=xx:xx:xx:xx:xx
tenancy=ocid1.tenancy.oc1..xxxxxx
region=ap-singapore-1
key_file=C:\Users\你的帳號\Desktop\私密金鑰.pem
```

---

## GitHub Actions 自動搶機（推薦）

不需要電腦一直開著，24 小時不間斷自動重試。

### 運作方式

```
Job 開始
  └─ 每 90 秒嘗試一次，持續約 350 分鐘（~233 次）
       ├─ 搶到了 → 自動停用 workflow，結束
       └─ timeout → 立刻觸發下一個 Job，空窗幾乎為零
                        └─ 重複循環...

備用：每 12 小時 cron 補觸發（防止接力鏈條意外中斷）
```

| 項目 | 說明 |
|------|------|
| 每次 Job 持續時間 | 約 350 分鐘（~233 次嘗試）|
| 接力方式 | Job 結束時自動觸發下一個，空窗幾乎為零 |
| 同時執行上限 | 最多 1 個跑 + 1 個排隊（不會累積）|
| 備用排程 | 每 12 小時 cron（鏈條斷掉時自動恢復）|
| 搶到後 | workflow 自動停用，完全不需要手動操作 |
| ARM 與 Micro | 各自獨立 workflow，可同時執行互不干擾 |

---

## 設定步驟

### Step 1：建立 GitHub 倉庫

建立 **public** 倉庫（public 才有無限制的 Actions 分鐘數）。

### Step 2：產生 OCI API 金鑰

1. Oracle Cloud 主控台 → 右上角頭像 → **我的設定檔**
2. 左側選單 → **API 金鑰** → **新增 API 金鑰**
3. 選「**產生 API 金鑰對**」→ 下載私密金鑰（`.pem` 檔）
4. 按「新增」後複製出現的設定文字，從中取得以下值：
   - `user`、`fingerprint`、`tenancy`、`region`

### Step 3：建立 GitHub Personal Access Token（PAT）

PAT 讓 workflow 搶到機器後能自動停用自己、或接力觸發下一個 Job。

1. GitHub → 右上角頭像 → **Settings**
2. 左側最下方 → **Developer settings**
3. **Personal access tokens** → **Tokens (classic)**
4. **Generate new token (classic)**
5. 填寫：
   - **Note**：隨意命名，例如 `oci-retry`
   - **Expiration**：依需求設定（建議 90 天或 No expiration）
   - **Scopes**：勾選 **`workflow`**（必須勾，才能操作 workflow）
6. 點 **Generate token** → **立刻複製**產生的 token（只顯示一次）

### Step 4：設定 GitHub Secrets

到倉庫 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

共需設定 **6 個** secret：

| Secret 名稱 | 填入的值 | 取得位置 |
|------------|---------|----------|
| `OCI_USER` | `ocid1.user.oc1..xxxxxx` | OCI 主控台 → 我的設定檔 → OCID |
| `OCI_FINGERPRINT` | `xx:xx:xx:xx:xx` | OCI 主控台 → 我的設定檔 → API 金鑰 → 指紋 |
| `OCI_TENANCY` | `ocid1.tenancy.oc1..xxxxxx` | OCI 主控台 → 我的設定檔 → 租用戶 OCID |
| `OCI_REGION` | 例如 `ap-singapore-1` | OCI 主控台右上角區域名稱 |
| `OCI_PRIVATE_KEY` | `.pem` 檔案完整內容 | 開啟 Step 2 下載的 `.pem` 檔，複製全部文字（含頭尾 `-----BEGIN/END PRIVATE KEY-----`）|
| `GH_PAT` | GitHub Personal Access Token | Step 3 產生的 token |

### Step 5：推送程式碼到 GitHub

```bash
git remote add origin https://github.com/你的帳號/你的倉庫名稱.git
git branch -M main
git push -u origin main
```

### Step 6：啟動 workflow

1. 到倉庫 → **Actions** 頁籤
2. 若 workflow 顯示停用，點 **Enable workflow**
3. 點 **Run workflow** 手動立刻觸發第一次

ARM 和 Micro 各自有獨立的 workflow，可以同時啟動。

---

## 成功後

搶到 instance 後，workflow 會**自動停用**，不需要任何手動操作。

到 Oracle Cloud 主控台 → **執行處理** → 查看公用 IP，即可 SSH 連線：

```bash
ssh -i 私密金鑰.pem ubuntu@公用IP
```

---

## 注意事項

- `COMPARTMENT_ID` 和 `SSH_PUBLIC_KEY` 寫在腳本裡是安全的（非機密資訊）
- OCI API 私密金鑰、PAT 等敏感資訊全部放在 GitHub Secrets，不進版本控制
- `.pem` / `.key` / `.pub` 已加入 `.gitignore`，不會被 commit
