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

## Preventing repeats

- Run a pre-commit secret scanner locally: [gitleaks](https://github.com/gitleaks/gitleaks) or [pre-commit](https://pre-commit.com/) + [detect-secrets](https://github.com/Yelp/detect-secrets).
- Store credentials in a secret manager (Vault, AWS Secrets Manager, GitHub Actions secrets), not source files.
- Keep `.env` files in `.gitignore`; commit a `.env.example` template instead.
