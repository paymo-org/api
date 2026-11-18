# Secret detected in this branch

A secret-scanning workflow (gitleaks or TruffleHog) flagged a credential. **Treat the secret as already compromised** — once it has been pushed to GitHub, mirrors, forks, search indexers, and GitHub's own ~90-day commit cache may already hold a copy.

The steps below are ordered. Step 1 is the only one that actually closes the hole; the rest prevent re-discovery.

## 1. Rotate the credential — do this first

Revoke and reissue the credential at its source:

- API key / token → rotate in the provider's dashboard (AWS, GCP, Stripe, etc.)
- Database or service password → change it and roll any consumers
- SSH / deploy key → remove the public key, generate a new pair

Scrubbing history does **not** undo exposure. Rotation is what closes the hole.

If the credential isn't yours, notify the owning team before continuing.

## 2. Identify the offending commits

Open the workflow run linked from the PR comment. The scanner output names the detector, file path, and commit SHA(s).

**Do not paste the raw secret value into the PR thread, an issue, Slack, or any chat** — those surfaces are visible to read-access users and are hard to scrub later.

## 3. Remove the secret from history

Use [`git filter-repo`](https://github.com/newren/git-filter-repo) (the modern replacement for `git filter-branch`):

```bash
pip install git-filter-repo

# Work from a fresh clone — filter-repo refuses to run on a non-fresh clone.
git clone <repo-url> repo-clean
cd repo-clean
git checkout <branch>

# Option A — redact a specific string everywhere it appears in history.
#
# IMPORTANT: do NOT type the secret on the command line (e.g. via `printf` or
# `echo`) — it will land in your shell history and create a second leak.
# Instead, write replacements.txt with an editor and paste the value in:
#
#   $EDITOR replacements.txt
#
# File format (one rule per line, literal match):
#
#   <the-secret-value>==>REDACTED
#
# Then run:
git filter-repo --replace-text replacements.txt

# Option B — drop a file entirely from history:
git filter-repo --path path/to/file --invert-paths
```

`git filter-repo` rewrites every commit SHA on the affected branch.

## 4. Force-push the cleaned branch

```bash
git push --force-with-lease origin <branch>
```

Use `--force-with-lease`, not `--force` — it refuses to overwrite commits you haven't pulled, which protects against clobbering a teammate's work pushed in the meantime.

## 5. Tell collaborators to re-clone

Anyone with the branch checked out must:

1. Save any local work in progress.
2. Delete their local copy of the branch.
3. Re-clone or re-fetch.

A `git pull` or rebase is **not enough** — their reflog still contains the original commits with the secret.

## 6. Purge GitHub's cache (only if needed)

Force-pushed commits remain reachable by direct SHA URL on github.com for **~90 days** after the force-push. To remove them sooner, open a request with [GitHub Support](https://support.github.com/contact) and ask them to purge the cached commits.

For public repos, also assume external mirrors (GH Archive, code-search services, scrapers) hold copies. Step 1 (rotation) is your real defense.

## 7. Re-run the scan

After the force-push, the secret-scanning workflow re-runs automatically and the check should pass. Once it does, the PR is mergeable again.

## Alternative: suppress the dead finding without rewriting history

Steps 3–6 rewrite git history. That's disruptive — every collaborator must re-clone, every open PR rebases, and external mirrors keep the old commits anyway. If you've completed **Step 1 (rotation)** and the credential is confirmed dead, you can skip the history rewrite and add a narrow allowlist entry so the scanner stops re-reporting it.

This is a judgement call. Rewrite history if legal/compliance requires it, if the repo is private with a small team, or if the secret grants access to data that was sensitive at the time of exposure. Otherwise, suppression is usually the lower-cost path — and rotation is what actually closed the hole.

### Gitleaks — `.gitleaksignore` by fingerprint

Gitleaks emits a fingerprint per finding: `<commit>:<file>:<rule-id>:<line>`. The per-repo summary md in `security/run/<run>/<repo>/gitleaks-*.md` lists them in the Details table. Paste the exact fingerprints into a `.gitleaksignore` at the repo root, annotated with the revocation context:

```
# revoked 2026-05-12 — AWS key rotated in vault, ticket SEC-482
a1b2c3d4e5f6:infra/ci/deploy.sh:aws-access-token:42
a1b2c3d4e5f6:infra/ci/deploy.sh:aws-secret-key:43
```

For broader scope (same dead key across many files), prefer the `commits` allowlist in `.gitleaks.toml` — pin the dead commit SHA — over a regex. A regex that hides one dead AWS key will hide future live ones too.

### TruffleHog — `--exclude-paths` or rely on verification

TruffleHog has no `.trufflehogignore` file. Two options:

1. **Path-scoped:** add an `exclude.txt` listing file globs and pass `--exclude-paths=exclude.txt`. Best when the dead key sits in a known fixture or config file.
2. **Trust the verifier:** TruffleHog calls the provider's API to confirm the key still works. Once you've rotated, re-run with verification on — the finding flips from `verified=true` to `verified=false` in the per-repo summary. Workflows that gate on `--results=verified` then pass without any allowlist.

Re-run the scan after revocation and confirm the verifier sees the key as dead **before** adding an allowlist entry; otherwise you risk suppressing a finding that's still live.

### Allowlist hygiene

- Scope narrowly — fingerprint or specific path, never a broad regex.
- Annotate every entry with the date, who revoked it, and the ticket link.
- Commit the allowlist file alongside the rotation PR so the audit trail is in one place.
- Review the allowlist periodically; entries for keys no longer reachable in history can be removed.

## Preventing repeats

- Run a pre-commit secret scanner locally: [gitleaks](https://github.com/gitleaks/gitleaks) or [pre-commit](https://pre-commit.com/) + [detect-secrets](https://github.com/Yelp/detect-secrets).
- Store credentials in a secret manager (Vault, AWS Secrets Manager, GitHub Actions secrets), not source files.
- Keep `.env` files in `.gitignore`; commit a `.env.example` template instead.
