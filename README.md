# Fix: Storage Check for New Contracts

This PR fixes the `core-contracts-storage-check` and `diamond-storage-check` CI workflows in [ubiquity/ubiquity-dollar](https://github.com/ubiquity/ubiquity-dollar) to no longer fail when new contracts are added.

## Problem
When a new contract is added to the monorepo, the `foundry-storage-check` action fails with "No workflow run found with an artifact named..." because there is no previous storage layout to compare against for a brand new contract.

## Solution
Before running the storage check, we verify that the contract/library existed in the base branch (not newly added). If it's a new file, we skip the storage check since a new contract has no prior storage that could collide.

## Changes
1. **core-contracts-storage-check.yml**: Added `git show` check to skip newly added core contracts
2. **diamond-storage-check.yml**: Added `git show` check to skip newly added diamond libraries

## QA
The fix ensures the following scenarios:
- ✅ No storage updates → CI passing
- ✅ Storage update, no collision → CI passing  
- ✅ Storage update, collision → CI failing (correct behavior)
- ✅ New contract added → CI passing (was failing before)
