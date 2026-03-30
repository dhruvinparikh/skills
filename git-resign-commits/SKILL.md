# Re-sign Git Commits for GitHub Verified Signature Requirements

## Overview

This skill covers re-signing existing Git commits with SSH keys to satisfy GitHub's repository rule requiring verified commit signatures (GH013). It uses an interactive rebase with `--exec` to amend and sign commits in-place, with a tree-hash verification step to guarantee code integrity.

## When to Use

- GitHub rejects `git push` with: `GH013: Repository rule violations found` / `Commits must have verified signatures`
- You need to sign specific historical commits (not just the latest)
- Commits were made without `-S` flag or before signing was configured
- A branch has a mix of signed and unsigned commits that all need to be signed

## Prerequisites

1. **SSH signing key** exists locally (e.g., `~/.ssh/id_ed25519`)
2. **Key is registered on GitHub** as a **Signing Key** (not just Authentication):
   - Go to `https://github.com/settings/ssh/new`
   - Set **Key type** → **Signing Key**
   - Paste the public key
3. **Git signing config** is set:
   ```bash
   git config gpg.format ssh
   git config user.signingkey ~/.ssh/id_ed25519.pub
   git config commit.gpgsign true
   ```

## Workflow

### Step 1: Save the tree hash for verification

The tree hash is a content fingerprint of all files at HEAD. It does not change when only commit metadata (signatures, hashes) changes.

```bash
git rev-parse HEAD^{tree} > /tmp/pre-rebase-tree-hash
```

### Step 2: Identify the base commit

Find the commit **before** the first unsigned commit. All commits between this base and HEAD will be re-signed.

```bash
git log --oneline -20
# Identify the last commit that does NOT need re-signing
# Example: b1c5db0 is the base
```

To check if a commit is signed:

```bash
git cat-file -p <HASH> | grep gpgsig
# If no output → unsigned
```

### Step 3: Rebase with --exec to re-sign

```bash
GIT_SEQUENCE_EDITOR=: git rebase --exec "git commit --amend --no-edit -S" <BASE_COMMIT>
```

- `GIT_SEQUENCE_EDITOR=:` prevents the interactive editor from opening (accepts all picks as-is)
- `--exec` runs the amend+sign command after each commit is applied
- All commits from `<BASE_COMMIT>` (exclusive) to HEAD are re-signed

### Step 4: Verify code integrity

```bash
POST_TREE=$(git rev-parse HEAD^{tree})
PRE_TREE=$(cat /tmp/pre-rebase-tree-hash)
if [ "$PRE_TREE" = "$POST_TREE" ]; then
  echo "✅ VERIFIED: Code is identical."
else
  echo "❌ MISMATCH: Code changed!"
fi
```

### Step 5: Verify all commits are signed

```bash
git log --oneline <BASE_COMMIT>..HEAD | while read hash msg; do
  echo -n "$hash $msg → "
  git cat-file -p $hash | grep -q gpgsig && echo "SIGNED ✅" || echo "UNSIGNED ❌"
done
```

### Step 6: Force push

```bash
git push --force-with-lease origin <BRANCH>
```

Use `--force-with-lease` (not `--force`) to avoid overwriting any new remote commits.

## Key Details

| Item | Notes |
|------|-------|
| Code safety | Tree hash (`HEAD^{tree}`) is a cryptographic fingerprint of all file contents — if it matches, the code is identical |
| Hash changes | All rebased commit SHAs will change (expected, since parent hashes cascade) |
| Timestamps | Original author/committer dates are preserved |
| `--force-with-lease` | Safer than `--force` — aborts if remote has new commits you haven't fetched |

## Common Pitfalls

- **Key must be a Signing key on GitHub**, not just an Authentication key. Both can be added from the same SSH key, but they serve different purposes.
- **Email must match**: The commit author email must match a verified email on the GitHub account where the signing key is registered.
- **All downstream commits change**: If you re-sign commit C, then commits D, E, F... all get new hashes too (even if already signed), because each commit's hash depends on its parent.
- **`git verify-commit` may fail locally** even for properly signed commits if `gpg.ssh.allowedSignersFile` is not configured. GitHub does its own verification server-side — local verification failure does not mean the push will fail.
