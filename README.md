# OCI (Oracle Cloud Infrastructure) Free VM Grabber Script

## Two Types of Scripts

| Script | Target Instance Type | Specifications | Architecture |
|--------|---------------------|-----------------|--------------|
| `oci_retry.py` | VM.Standard.A1.Flex | 4 OCPU / 24 GB RAM / 200 GB Disk | ARM (Ampere) |
| `oci_retry_micro.py` | VM.Standard.E2.1.Micro | 1 OCPU / 1 GB RAM / 50 GB Disk | x86 |

Both are part of Oracle's Always Free perpetual free tier. Each account can have up to 4 OCPU ARM (can be split into multiple instances) + 2 Micro instances.

---

## Script Configuration

Open the script and modify the two variables at the top:

```python
COMPARTMENT_ID = "ocid1.tenancy.oc1..your_OCID"
SSH_PUBLIC_KEY = "ssh-rsa AAAA...your_public_key_content"
RETRY_INTERVAL = 90  # Retry interval (seconds)
```

### Getting Compartment ID (Tenancy OCID)

**Purpose**: Tell Oracle which account to create resources under

1. Oracle Cloud Console → Avatar (top right) → **My Profile**
2. Or: Menu (top left) → **Identity & Security** → **Compartments** → Click root
3. Copy the OCID (format: `ocid1.tenancy.oc1..xxxxxx`)

### Getting SSH Public Key

**Purpose**: Used for SSH connection after the instance is created

1. When creating an instance in Oracle Cloud, select "**Generate Key Pair**"
2. Download two files:
   - `*.key` or `*.pem` (Private key, keep it safe, used for connection)
   - `*.pub` (Public key, paste into script)
3. Open the `.pub` file with a text editor, copy all content (format starts with `ssh-rsa AAAA...`)

Or generate your own using `ssh-keygen` on your local machine.

---

## Running Locally

```bash
# Install package (only once)
pip install oci

# Windows: Navigate to folder
cd /d D:\Projects\oci_arm_retry

# Run ARM version
python oci_retry.py

# Run Micro version
python oci_retry_micro.py
```

### Local Execution Requires OCI Config File

**Location**: `C:\Users\your_account\.oci\config` (create the folder and file, no file extension)

```
[DEFAULT]
user=ocid1.user.oc1..xxxxxx
fingerprint=xx:xx:xx:xx:xx
tenancy=ocid1.tenancy.oc1..xxxxxx
region=ap-singapore-1
key_file=C:\Users\your_account\Desktop\private_key.pem
```

**Getting Config File Content**:
1. Oracle Cloud Console → Avatar (top right) → **My Profile**
2. Left menu → **API Keys** → **Add API Key**
3. Select "**Generate API Key Pair**" → Download private key (`.pem` file)
4. Click "Add" → Copy the configuration text that appears, paste into `config` file
5. Change the `key_file` path to the actual location of your `.pem` file

### Script Execution Behavior

| Status | Description |
|--------|-------------|
| Initialization | Automatically creates VCN + subnet (skips if exists) |
| Grab Loop | Attempts once every `RETRY_INTERVAL` seconds |
| Capacity Unavailable | Prints ❌ and continues retrying |
| Network Timeout | Prints ⚠️ and continues retrying (won't crash) |
| Success | Prints ✅ instance ID and stops automatically |

---

## GitHub Actions Auto-Grab (Recommended)

No need to keep your computer on. Retries automatically 24/7.

### How It Works

```
Job starts
  └─ Attempts every 90 seconds, runs for ~350 minutes (~233 attempts)
       ├─ Success → Automatically disables workflow, done
       └─ timeout → Immediately triggers next Job, almost zero downtime
                        └─ Repeats...

Chain breaks: Manually click "Run workflow" in Actions tab
```

| Item | Description |
|------|-------------|
| Job Duration | ~350 minutes (~233 attempts) |
| Relay Method | Next job triggers when current ends, almost zero downtime |
| Concurrent Limit | Max 1 running + 1 queued (no accumulation) |
| Chain Breaks | Manually click "Run workflow" in Actions tab |
| After Success | Workflow automatically disables, no manual action needed |
| ARM & Micro | Independent workflows, can run simultaneously without interference |

---

## Configuration Steps

### Step 1: Create GitHub Repository

Create a **public** repository (public has unlimited Actions minutes).

### Step 2: Generate OCI API Key

1. Oracle Cloud Console → Avatar (top right) → **My Profile**
2. Left menu → **API Keys** → **Add API Key**
3. Select "**Generate API Key Pair**" → Download private key (`.pem` file)
4. Click "Add" → Copy the configuration text, extract these values:
   - `user`, `fingerprint`, `tenancy`, `region`

### Step 3: Create GitHub Personal Access Token (PAT)

PAT allows the workflow to automatically disable itself after grabbing an instance or trigger the next job.

1. GitHub → Avatar (top right) → **Settings**
2. Bottom of left menu → **Developer settings**
3. **Personal access tokens** → **Tokens (classic)**
4. **Generate new token (classic)**
5. Fill in:
   - **Note**: Name it anything, e.g., `oci-retry`
   - **Expiration**: Set as needed (recommend 90 days or No expiration)
   - **Scopes**: Check **`workflow`** (required to manage workflows)
6. Click **Generate token** → **Copy immediately** (shown only once)

### Step 4: Configure GitHub Secrets

Go to Repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Need to set **6** secrets total:

| Secret Name | Value | Where to Get |
|-------------|-------|--------------|
| `OCI_USER` | `ocid1.user.oc1..xxxxxx` | OCI Console → My Profile → OCID |
| `OCI_FINGERPRINT` | `xx:xx:xx:xx:xx` | OCI Console → My Profile → API Keys → Fingerprint |
| `OCI_TENANCY` | `ocid1.tenancy.oc1..xxxxxx` | OCI Console → My Profile → Tenancy OCID |
| `OCI_REGION` | e.g., `ap-singapore-1` | OCI Console, region name (top right) |
| `OCI_PRIVATE_KEY` | Full `.pem` file content | Open `.pem` from Step 2, copy all text (including `-----BEGIN/END PRIVATE KEY-----`) |
| `GH_PAT` | GitHub Personal Access Token | Token from Step 3 |

### Step 5: Push Code to GitHub

```bash
git remote add origin https://github.com/your_account/your_repo_name.git
git branch -M main
git push -u origin main
```

### Step 6: Start Workflow

1. Go to Repository → **Actions** tab
2. If workflow shows as disabled, click **Enable workflow**
3. Click **Run workflow** to trigger the first run manually

ARM and Micro each have independent workflows and can be started simultaneously.

---

## After Success

After grabbing an instance, the workflow will **automatically disable** with no manual action needed.

Go to Oracle Cloud Console → **Compute Instances** → Check public IP and SSH:

```bash
ssh -i private_key.pem ubuntu@public_IP
```

---

## Important Notes

- `COMPARTMENT_ID` and `SSH_PUBLIC_KEY` in the script are safe (non-sensitive)
- OCI API private key, PAT, and other sensitive info go in GitHub Secrets, not version control
- `.pem` / `.key` / `.pub` are already in `.gitignore`, won't be committed
