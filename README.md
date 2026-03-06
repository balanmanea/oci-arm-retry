# OCI ARM Instance 搶機腳本設定說明

## 腳本需要的三樣資訊

### 1. Compartment ID（租用戶 OCID）

**用途**：告訴 Oracle 要在哪個帳號下建立資源

**取得方式**：
1. Oracle Cloud 主控台 → 右上角頭像 → **我的設定檔**
2. 或：左上角選單 → **身份識別與安全** → **區間** → 點 root
3. 複製 OCID（格式：`ocid1.tenancy.oc1..xxxxxx`）

---

### 2. SSH 公開金鑰

**用途**：建立 instance 後用來 SSH 連線進入 server

**取得方式**：
1. 在 Oracle Cloud 建立 instance 時，選「**產生 API 金鑰對**」
2. 下載兩個檔案：
   - `*.key` 或 `*.pem`（私密金鑰，自己保管，用來連線）
   - `*.pub`（公開金鑰，這個填入腳本）
3. 用記事本開啟 `.pub` 檔，複製全部內容（格式：`ssh-rsa AAAA...` 開頭）

---

### 3. OCI 設定檔（`~/.oci/config`）

**用途**：腳本透過 `oci.config.from_file()` 自動讀取，用於 API 認證

**取得方式**：
1. Oracle Cloud 主控台 → 右上角頭像 → **我的設定檔**
2. 左側選單 → **API 金鑰** → **新增 API 金鑰**
3. 選「**產生 API 金鑰對**」→ 下載私密金鑰（`.pem` 檔）
4. 按「新增」後會出現設定文字，複製起來

**儲存位置**：`C:\Users\你的帳號\.oci\config`（建立資料夾和檔案，無副檔名）

**設定檔格式**：
```
[DEFAULT]
user=ocid1.user.oc1..xxxxxx
fingerprint=xx:xx:xx:xx:xx
tenancy=ocid1.tenancy.oc1..xxxxxx
region=ap-singapore-1
key_file=C:\Users\你的帳號\Desktop\私密金鑰.pem
```

---

## 腳本設定位置

開啟 `oci_retry.py`，修改最上方的兩個變數：

```python
COMPARTMENT_ID = "ocid1.tenancy.oc1..你的OCID"
SSH_PUBLIC_KEY = "ssh-rsa AAAA...你的公開金鑰內容"
RETRY_INTERVAL = 90  # 重試間隔（秒），建議 60-120
```

---

## 執行方式

```bash
# 安裝套件（只需一次）
pip install oci

# 切換到腳本資料夾
cd D:\Projects\oci_arm_retry

# 執行
python oci_retry.py
```

---

## 腳本行為說明

| 步驟 | 說明 |
|------|------|
| 初始化 | 自動建立 VCN + 子網路（已存在則跳過）|
| 搶機循環 | 每隔 `RETRY_INTERVAL` 秒嘗試一次 |
| 容量不足 | 印出 ❌ 並繼續重試 |
| 網路逾時 | 印出 ⚠️ 並繼續重試（不會當掉）|
| 成功 | 印出 ✅ instance ID 並自動停止 |

**建立的 instance 規格**：VM.Standard.A1.Flex，4 OCPU，24 GB RAM，200 GB 磁碟（全部在 Always Free 免費範圍內）

---

## 注意事項

- 終端機不能關，讓腳本一直在背景跑
- 成功後到 Oracle Cloud 主控台 → **執行處理** 查看公用 IP
- 如果懷疑腳本在成功後沒停下來，手動 Ctrl+C 並去主控台確認是否已建立

---

## GitHub Actions 自動搶機設定

用 GitHub Actions 代替本機終端機，24 小時不間斷自動重試，不需要電腦一直開著。

### 運作方式

| 項目 | 說明 |
|------|------|
| 實際重試頻率 | Job 內每 90 秒試一次，持續 350 分鐘（約 233 次）|
| cron 排程 | 每 5 分鐘觸發一次 |
| 同時執行上限 | 最多 1 個跑 + 1 個排隊（concurrency 控制，不會累積）|
| 空窗 | Job 結束後排隊中的立刻接上，近乎零空窗 |
| 成功後 | 腳本自動退出，需手動關閉 workflow 停止排程 |
| 失敗/容量不足 | 持續重試直到 Job 結束，排隊中的 Job 立刻接手 |

### Step 1：建立 GitHub 倉庫

1. 到 [github.com](https://github.com) 建立新的 **public** 倉庫（public 才有無限制的 Actions 分鐘）
2. 把這個資料夾推上去（`oci_retry.py` + `.github/` 資料夾）

### Step 2：設定 GitHub Secrets

到倉庫 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**，新增以下 5 個 secret：

| Secret 名稱 | 對應值 | 取得位置 |
|------------|--------|----------|
| `OCI_USER` | `ocid1.user.oc1..xxxxxx` | OCI 主控台 → 頭像 → 我的設定檔 → OCID |
| `OCI_FINGERPRINT` | `xx:xx:xx:xx:xx` | OCI 主控台 → 我的設定檔 → API 金鑰 → 指紋 |
| `OCI_TENANCY` | `ocid1.tenancy.oc1..xxxxxx` | OCI 主控台 → 頭像 → 我的設定檔 → 租用戶 OCID |
| `OCI_REGION` | `ap-singapore-1` | 看你的帳號區域（新加坡填此值）|
| `OCI_PRIVATE_KEY` | PEM 檔案的完整內容 | 開啟下載的 OCI API 私密金鑰 `.pem` 檔，複製全部文字（含頭尾 `-----BEGIN/END PRIVATE KEY-----`）|

> `OCI_PRIVATE_KEY` 的值要包含 `-----BEGIN PRIVATE KEY-----` 到 `-----END PRIVATE KEY-----` 的完整內容。

### Step 3：啟動 workflow

1. 推送程式碼到 GitHub 後，到倉庫 → **Actions** 頁籤
2. 若 workflow 被停用，點 **Enable workflow**
3. 可以點 **Run workflow** 手動立刻觸發一次測試

### Step 4：成功後關閉

搶到 instance 後：
1. 到 **Actions** 頁籤 → 左側選 `OCI ARM 搶機`
2. 點右側 `...` → **Disable workflow**（停止排程）
3. 到 Oracle Cloud 主控台確認 instance 已建立並取得公用 IP
