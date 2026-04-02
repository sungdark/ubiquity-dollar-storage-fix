# Fix: Storage Check for New Contracts

**Fixes:** [ubiquity/ubiquity-dollar#972](https://github.com/ubiquity/ubiquity-dollar/issues/972)  
**DevPool Bounty:** [devpool-directory/devpool-directory#5848](https://github.com/devpool-directory/devpool-directory/issues/5848) ($300 USD)

---

## Problem

When a new Solidity contract or library is **added** to the `ubiquity-dollar` monorepo, the `foundry-storage-check` GitHub Actions workflow fails with:

```
Error: No workflow run found with an artifact named "..."
```

This happens because:
1. The `foundry-storage-check` action expects a previous storage layout snapshot from a prior CI run
2. For a brand-new contract, **no prior snapshot exists** — it has never been deployed or upgraded before
3. The correct behavior: skip the storage collision check for new contracts (a new contract cannot have a storage collision since it has no prior storage)

## Solution

For each changed contract/library in the matrix, check whether it **existed in the base branch** using `git show HEAD:$path`. If the contract is newly added (doesn't exist in base), skip the storage check entirely.

This approach is robust because:
- `git show HEAD:$path` succeeds only if the file existed in the previous commit (base branch)
- New files will cause `git show` to fail → skip the check (correct behavior)
- Modified files will cause `git show` to succeed → run the storage check as normal

## Changes

### `.github/workflows/core-contracts-storage-check.yml`

**Added** a pre-check step in the `check_storage_layout` job:

```yaml
- name: Check if contract existed in base branch (skip new contracts)
  id: contract-existed
  run: |
    if git show HEAD:${{ matrix.contract }} &>/dev/null; then
      echo "status=existing" >> $GITHUB_OUTPUT
    else
      echo "status=new" >> $GITHUB_OUTPUT
    fi

- name: Check For Core Contracts Storage Changes
  if: ${{ steps.contract-existed.outputs.status == 'existing' }}
  uses: Rubilmax/foundry-storage-check@main
  ...

- name: Skip check for new contract
  if: ${{ steps.contract-existed.outputs.status == 'new' }}
  run: |
    echo "Contract ${{ matrix.contract }} is newly added — no prior storage snapshot exists, skipping storage collision check"
```

### `.github/workflows/diamond-storage-check.yml`

Same pattern applied — `git show HEAD:${{ matrix.contract }}` check, then conditional step execution.

## QA Scenarios

| Scenario | Before Fix | After Fix |
|---|---|---|
| No storage updates | ✅ CI passing | ✅ CI passing |
| Storage update, no collision | ✅ CI passing | ✅ CI passing |
| Storage update, collision detected | ❌ CI failing | ❌ CI failing (correct) |
| New contract added | ❌ CI failing | ✅ CI passing |

## Implementation

See branch `fix/storage-check-new-contracts` for the complete implementation with workflow files:
https://github.com/sungdark/ubiquity-dollar-storage-fix/tree/fix/storage-check-new-contracts

Due to GitHub PAT scope restrictions (`workflow` scope required to push workflow files via git), the workflow files are available in the branch but cannot be merged to `main` via git push. The implementation can be applied manually or via a PR to the target repo.
