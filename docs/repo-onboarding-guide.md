# Repo Onboarding Guide — 接入 shared-workflows

將新 repo 接入 Chao-hu-Lab/shared-workflows 的標準流程。
基於 ms-preprocessing-toolkit 的實際遷移經驗撰寫。

---

## 前置條件

| 項目 | 說明 |
|------|------|
| Repo 位置 | 已轉移至 `Chao-hu-Lab` org |
| Repo 可見性 | **Public**（免費方案下 public repo 才有完整 Actions 功能） |
| Self-hosted runner | Org-level runner 已註冊且服務運行中 |
| Runner group | Default group 已開啟 **Allow public repositories** |

---

## Step 1：更新 remote URL

Repo 從個人帳號轉移到 org 後，GitHub 會 redirect，但 redirect 不是永久保證。

```powershell
# 主 repo
git remote set-url origin https://github.com/Chao-hu-Lab/<repo-name>.git

# 若有 submodule（例如 ms-core）
git config --file .gitmodules submodule.<name>.url https://github.com/Chao-hu-Lab/<submodule-name>.git
git submodule sync
git add .gitmodules
git commit -m "chore: update remote URLs to Chao-hu-Lab org"
```

**驗證：** `git fetch origin --dry-run` 不應出現 `This repository moved` 警告。

---

## Step 2：新增 ci.yml

刪除舊的 CI workflow 內容，替換為：

### A. pyproject.toml 型 repo（有 `.[dev]` extras）

```yaml
name: CI

on:
  push:
    branches: ["master", "main", "feature/*", "fix/*"]
  pull_request:
    branches: ["master", "main"]

jobs:
  test:
    uses: Chao-hu-Lab/shared-workflows/.github/workflows/python-ci.yml@main
    with:
      python-versions: '["3.11", "3.12"]'
      pythonpath: ""          # 有 submodule 時填 "ms-core/src" 等
      submodules: false       # 有 submodule 時改 true
```

### B. requirements.txt 型 repo（傳統專案）

```yaml
name: CI

on:
  push:
    branches: ["master", "main", "feature/*", "fix/*", "refactor/*"]
  pull_request:
    branches: ["master", "main"]

jobs:
  test:
    uses: Chao-hu-Lab/shared-workflows/.github/workflows/python-ci.yml@main
    with:
      python-versions: '["3.11", "3.12"]'
      install-args: "-r requirements.txt pytest"  # ← 關鍵差異：需含 pytest
      pythonpath: "src"
      submodules: false
```

### 參數對照

| 參數 | 用途 | 預設值 | 範例 |
|------|------|--------|------|
| `python-versions` | matrix 版本 | `'["3.11", "3.12"]'` | — |
| `install-args` | uv pip install 參數 | `"-e .[dev]"` | `"-r requirements.txt"` |
| `pythonpath` | 測試時的 PYTHONPATH | `""` | `"src"`, `"ms-core/src"` |
| `submodules` | 是否 checkout submodule | `false` | `true` / `false` |
| `test-args` | 額外 pytest 參數 | `"-v --tb=short -x"` | `"--tb=long"` |

---

## Step 3：新增 build.yml

### A. pyproject.toml 型 repo

```yaml
name: Build Desktop Packages

on:
  push:
    tags: ["v*.*.*"]
  workflow_dispatch:
    inputs:
      upload_artifact:
        description: "Upload as artifact (for testing without a tag)"
        type: boolean
        default: true

permissions:
  contents: write      # ← 必須在最外層，不能放在 job 層級

jobs:
  build:
    uses: Chao-hu-Lab/shared-workflows/.github/workflows/python-build.yml@main
    with:
      upload_artifact: ${{ inputs.upload_artifact || false }}
      spec-file: "<your-app>.spec"
      executable-name: "<your-app>"
      artifact-prefix: "<your-app>"
      pythonpath: "ms-core/src"                                          # 有 submodule 時
      verify-command: "{exe} --version"
      extra-package-files: '[{"src":"docs/release/README.md","dst":"README.md"},{"src":"LICENSE","dst":"LICENSE"}]'
```

### B. requirements.txt 型 repo

```yaml
name: Build Desktop Packages

on:
  push:
    tags: ["v*.*.*"]
  workflow_dispatch:
    inputs:
      upload_artifact:
        description: "Upload as artifact (for testing without a tag)"
        type: boolean
        default: true

permissions:
  contents: write

jobs:
  build:
    uses: Chao-hu-Lab/shared-workflows/.github/workflows/python-build.yml@main
    with:
      upload_artifact: ${{ inputs.upload_artifact || false }}
      spec-file: "build/<your-app>.spec"
      executable-name: "<your-app>"
      artifact-prefix: "<your-app>"
      install-command: "pip install -r requirements.txt pyinstaller"     # ← 關鍵差異
      pythonpath: "src"
      version-source: "tag"                                              # ← 無 pyproject.toml
      extra-package-files: '[{"src":"README.md","dst":"README.md"}]'
```

### Build 參數對照

| 參數 | 用途 | 預設值 |
|------|------|--------|
| `spec-file` | PyInstaller spec 檔路徑 | `"ms-preprocessing.spec"` |
| `executable-name` | 產出執行檔名（不含副檔名） | `"ms-preprocessing"` |
| `artifact-prefix` | artifact zip 前綴 | `"ms-preprocessing"` |
| `install-command` | 安裝指令 | `"pip install -e . pyinstaller"` |
| `pythonpath` | 建置用 PYTHONPATH | `""` |
| `version-source` | 版號來源：`pyproject` 或 `tag` | `"pyproject"` |
| `platforms` | 建置平台 JSON | `'["windows", "macos"]'` |
| `extra-package-files` | 額外打包檔案 JSON | `'[]'` |
| `verify-command` | 驗證指令（`{exe}` 為佔位符） | `""`（跳過） |

> **注意：** `permissions: contents: write` 在 `workflow_call` 架構下必須放在 caller workflow 的最外層，放在 job 層級會被忽略。

---

## Step 4：新增 dependabot.yml

建立 `.github/dependabot.yml`：

```yaml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    labels: ["dependencies"]

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels: ["dependencies"]
```

---

## Step 5：更新測試

如果原本的測試有斷言 workflow 檔案內容（例如檢查 `Copy-Item` 命令），需要更新為驗證 shared-workflows delegation：

```python
# 舊：直接檢查打包指令
assert 'Copy-Item "docs\\release\\README.md"' in workflow

# 新：檢查 delegation
assert "Chao-hu-Lab/shared-workflows/.github/workflows/python-build.yml@main" in workflow
```

---

## Step 6：驗證 CI

1. Commit 所有變更並 push 到 feature branch
2. 確認 CI 觸發並全綠
3. 記下 status check 名稱（通常是 `test / Test (Python 3.11)` 和 `test / Test (Python 3.12)`）

```powershell
gh run list --repo Chao-hu-Lab/<repo-name> --limit 1
gh pr checks <pr-number> --repo Chao-hu-Lab/<repo-name>
```

---

## Step 7：設定 Branch Protection Rules

GitHub → repo Settings → Rules → Rulesets → Add ruleset

| 設定 | 值 |
|------|---|
| Ruleset name | `protect-master` |
| Target branches | Include default branch |
| Restrict deletions | ✅ |
| Block force pushes | ✅ |
| Require a pull request before merging | ✅（Required approvals = 0） |
| Require status checks to pass | ✅（填入 Step 6 記下的 check 名稱） |

> **Status checks 必須等 CI 至少跑過一次**，GitHub UI 才能搜尋到 check 名稱。

---

## Troubleshooting：實際踩過的坑

### 1. Runner 不接 job（Idle 狀態）

**症狀：** Workflow 顯示 `Workflow runs completed with no jobs`，runner 顯示 Idle。

**原因：** Org runner group 的 **Allow public repositories** 沒開。Public repo 的 job 不會被分配給不允許 public 的 runner group。

**修復：** GitHub → Org Settings → Actions → Runner groups → Default → 勾選 Allow public repositories。

---

### 2. `shell: bash` 失敗（Windows self-hosted）

**症狀：** `bash: command not found` 或 composite action 步驟報錯。

**原因：** Self-hosted Windows runner 以 Windows 服務運行，服務帳號的 PATH 不包含 Git Bash。GitHub-hosted runner 會自動設定 bash，但 self-hosted 不會。

**修復：** 所有 shared-workflows 的 `shell:` 統一用 `pwsh`：
```yaml
shell: pwsh    # ✅ Windows 服務帳號一定有
shell: bash    # ❌ 可能找不到
```

---

### 3. `uv pip install --system` 報 externally managed

**症狀：**
```
error: The interpreter at ...uv/python/cpython-3.11... is externally managed
hint: Virtual environments were not considered due to the `--system` flag
```

**原因：** `astral-sh/setup-uv@v5` 安裝的是 uv 管理的 Python，會標記為 externally managed，禁止 `--system` 直接裝套件。

**修復：** 改用 venv：
```yaml
# setup-python/action.yml
- name: Install dependencies
  shell: pwsh
  run: |
    uv venv
    uv pip install ${{ inputs.install-args }}
```

對應的測試步驟也要改用 `uv run`：
```yaml
# python-ci.yml
- name: Run tests
  shell: pwsh
  run: uv run pytest tests/ ${{ inputs.test-args }}
```

---

### 4. Runner label 大小寫不符

**症狀：** Job 排隊但不執行，runner 卻是 Idle。

**原因：** GitHub Actions runner labels **區分大小寫**。Runner 上是 `Windows`（大寫 W），workflow 寫了 `windows`（小寫 w）。

**修復：** 對齊 label：
```yaml
runs-on: [self-hosted, Windows, X64]    # ✅ 與 runner 一致
runs-on: [self-hosted, windows]          # ❌ 大小寫不符
```

**確認方式：** GitHub → Org Settings → Actions → Runners → 查看 runner 的 Labels。

---

### 5. `permissions` 放錯位置

**症狀：** Release job 無法建立 GitHub Release（403 permission denied）。

**原因：** `permissions: contents: write` 放在 job 層級，但 `workflow_call` 架構下 permissions 必須在 caller 的最外層。

**修復：**
```yaml
# ✅ 正確：放在 workflow 最外層
permissions:
  contents: write

jobs:
  build:
    uses: ...

# ❌ 錯誤：放在 job 層級
jobs:
  build:
    permissions:
      contents: write
    uses: ...
```

---

### 6. 測試的 SimpleNamespace 缺少新增欄位

**症狀：** `AttributeError: 'types.SimpleNamespace' object has no attribute 'xxx'`，測試 return code 為 1。

**原因：** 程式碼新增了 CLI 參數（如 `export_deleted_feature`），但測試中的 `_make_cli_args` helper 沒有同步更新。

**預防：** 每次新增 CLI 參數時，搜尋所有使用 `SimpleNamespace` 模擬 CLI args 的測試：
```powershell
grep -rn "SimpleNamespace" tests/ --include="*.py"
```

---

## 快速參考：最終檔案結構

```
<your-repo>/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml          # ~15 行，呼叫 shared-workflows
│   │   └── build.yml       # ~25 行，呼叫 shared-workflows
│   └── dependabot.yml      # pip + github-actions weekly
├── .gitmodules             # URL 指向 Chao-hu-Lab（若有 submodule）
└── ...
```
