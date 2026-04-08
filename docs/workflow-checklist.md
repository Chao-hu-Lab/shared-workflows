# GitHub Actions Workflow Checklist

每次新增或修改 workflow 時，對照此清單。

## 通用

- [ ] `on:` 觸發條件正確？（push branches, pull_request, workflow_call）
- [ ] `permissions:` 設為最小權限？（預設 read，write 只在需要時加）
- [ ] submodules 是否需要 `submodules: recursive`？
- [ ] 是否需要設定 `PYTHONPATH`？
- [ ] Actions 版本使用 pinned tag（`@v4`, `@v5`），不用 `@main` 或 `@latest`？
- [ ] matrix jobs 有 `fail-fast: false`？

## CI Workflow（呼叫 python-ci.yml）

- [ ] runner 使用 `[self-hosted, windows]`？
- [ ] `python-versions` 傳入正確的 JSON array？
- [ ] `pythonpath` 有填入（若有 submodule src 需要加入 path）？
- [ ] `submodules: true` 有設定（若有 submodule）？

## Build Workflow（呼叫 python-build.yml）

- [ ] tag 格式 `v*.*.*` 與 `pyproject.toml` 版本一致？
- [ ] `workflow_dispatch` 手動觸發有加？
- [ ] `spec-file`、`executable-name`、`artifact-prefix` 是否正確？
- [ ] 頂層 `permissions: contents: write` 有設定？

## Dependabot

- [ ] 包含 `pip` ecosystem？
- [ ] 包含 `github-actions` ecosystem？
- [ ] schedule 是 `weekly`？

## Branch Protection（GitHub UI 設定，每個 repo 各做一次）

- [ ] Restrict deletions ✅
- [ ] Block force pushes ✅
- [ ] Require pull request before merging ✅（Required approvals = 0）
- [ ] Require status checks to pass ✅（CI 第一次跑完後補上 check 名稱）
