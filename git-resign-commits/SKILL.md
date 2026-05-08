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

### Step 6: Push

Two options depending on the team's policy:

**A. Force-push to the same branch** (if allowed):

```bash
git push --force-with-lease origin <BRANCH>
```

Use `--force-with-lease` (not `--force`) to abort if remote has new commits you haven't fetched.

**B. Push to a new branch** (if force-push is disallowed or undesirable):

```bash
git branch -m <BRANCH> <BRANCH>-resigned       # rename locally
git push -u origin <BRANCH>-resigned           # push as new branch
# Open a fresh PR from <BRANCH>-resigned; close the old PR.
```

The old branch on origin stays untouched, and the new branch has signed commits with the identical tree.

## Alternative Workflow: Cherry-pick onto a Fresh Branch

If `git rebase --exec` fails with phantom "Your local changes would be overwritten" errors (e.g., from autocrlf, line-ending normalization, or merge commits in the range), use cherry-pick instead. This rebuilds the branch one commit at a time on a clean base:

```bash
# 1. Backup
git tag backup/<BRANCH>-pre-resign HEAD

# 2. Save tree hash for verification
git rev-parse HEAD^{tree} > /tmp/pre-resign-tree

# 3. List unsigned commits in chronological order (oldest first)
git log --pretty=format:'%H %G? %s' origin/main..<BRANCH>
#   N = no signature, G = good, E = expired, B = bad

# 4. Start a clean branch from the target base
git checkout origin/main
git switch -c <BRANCH>-resigned

# 5. Cherry-pick each commit (oldest → newest); commits are signed via commit.gpgsign=true
git cherry-pick <oldest-hash>
# ... repeat for each, in order ...
git cherry-pick <newest-hash>

# Conflicts: resolve, then `git cherry-pick --continue`
# Merge commits in the original range: skip them — origin/main already contains
# whatever was merged in. Cherry-pick only the non-merge commits.

# 6. Verify trees match
[ "$(git rev-parse HEAD^{tree})" = "$(cat /tmp/pre-resign-tree)" ] \
  && echo TREES_MATCH || echo TREES_DIFFER

# 7. Verify all commits signed
git log --pretty=format:'%h %G? %s' origin/main..HEAD

# 8. Push (option A or B above)
```

**When to prefer cherry-pick over rebase --exec:**

- The original branch contains merge commits you want to flatten.
- `git rebase` repeatedly fails with "local changes would be overwritten" despite a clean working tree (typically a `core.autocrlf=true` interaction with the index).
- You want to drop specific commits or reorder while re-signing.

**When to prefer rebase --exec:**

- Linear history, no merges, no line-ending issues — it's faster and one command.

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
- **Hardware security keys (FIDO2 / `ed25519-sk`)** require a physical touch for *every* signature. A rebase across N commits = N touches. Cherry-pick has the same cost. Don't walk away from the keyboard mid-rebase.
- **Phantom "local changes would be overwritten"**: If `git rebase` aborts with this error on a clean working tree, suspect `core.autocrlf=true` re-normalizing line endings during checkout. Workarounds: `git checkout-index -fa && git update-index --refresh`, or fall back to the cherry-pick workflow above.
- **Annotated vs lightweight backup tags**: `git tag <name>` makes a lightweight tag (no signing). `git tag -a <name> -m "..."` makes an annotated tag, which the global `tag.gpgsign` config may try to sign — pass `-c tag.gpgsign=false` if you don't want a security-key touch for the backup tag.
