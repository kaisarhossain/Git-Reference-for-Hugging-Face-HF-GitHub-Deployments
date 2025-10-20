# 📘 README — Git Reference for Hugging Face (HF) & GitHub Deployments

**Author:** Mohammed Golam Kaisar Hossain Bhuyan
**Email:** kaisar.hossain@gmail.com

This document is a comprehensive reference for using Git with Hugging Face Spaces and GitHub. It includes step-by-step commands and safety notes for cleaning sensitive files (like `.env`), configuring remotes, deploying to Hugging Face, and reverting back to GitHub. Keep this file in your projects for a repeatable, secure workflow.

---

## 🔐 Important Warnings (Read First)

- **Never** commit secrets (API keys, tokens, passwords) to a repository, even temporarily. If you accidentally commit them, rotate the secrets immediately after cleanup.
- Commands that rewrite history (e.g., `git filter-repo`) will change commit SHAs. This usually requires force-pushing and coordination with collaborators.
- **Force-pushing** (`--force`) replaces remote history. Use it only when you understand the consequences or you're the only maintainer.
- Prefer using the official Hugging Face authentication methods (HF CLI login or Personal Access Tokens) rather than embedding tokens in commands or URLs.

---

## Table of Contents

1. [Global Git Setup & Identity](#global-git-setup--identity)
2. [Git LFS (Large File Storage)](#git-lfs-large-file-storage)
3. [Removing Sensitive Files from History (`.env` cleanup)](#removing-sensitive-files-from-history-env-cleanup)
4. [Untracking `.env`, Adding to `.gitignore`, and Committing Cleanup](#untracking-env-adding-to-gitignore-and-committing-cleanup)
5. [Switching Remotes: GitHub → Hugging Face Space (HF)](#switching-remotes-github--hugging-face-space-hf)
6. [Pushing to HF: Authentication Options & Examples](#pushing-to-hf-authentication-options--examples)
7. [Reverting Remote Origin: HF → GitHub](#reverting-remote-origin-hf--github)
8. [Useful Verification & Safety Commands](#useful-verification--safety-commands)
9. [Recommended Workflow Example (complete)](#recommended-workflow-example-complete)
10. [Appendix & Notes](#appendix--notes)

---

## 1️⃣ Global Git Setup & Identity

Set your global Git username and email so commits are properly attributed to you:

```bash
git config --global user.name "kaisarhossain"
git config --global user.email "kaisar.hossain@gmail.com"
```

**What this does:** Writes your name/email to `~/.gitconfig`. Commits you make on this machine will show this identity unless overridden per-repo.

---

## 2️⃣ Git LFS (Large File Storage)

If your project contains large binaries (models, datasets), use Git LFS — it keeps large files out of normal Git history while storing pointers instead.

```bash
# Initialize Git LFS for this machine (one-time per machine)
git lfs install

# Track a file pattern (example: model checkpoints)
git lfs track "*.pt"
git add .gitattributes
git commit -m "Track model files with Git LFS"
```

**What this does:** `git lfs install` configures Git on your machine to use LFS. `git lfs track` creates `.gitattributes` entries so file patterns use LFS storage instead of regular Git blob storage.

---

## 3️⃣ Removing Sensitive Files from History (`.env` cleanup)

If a `.env` file or other secrets were pushed, you must remove them from **all commits**. Use `git-filter-repo` — it is faster and safer than `git filter-branch`.

> **Install `git-filter-repo` (Python package)**
```bash
pip install --upgrade git-filter-repo
```

> **Rewrite history to remove `.env` entirely** (destructive—rewrites commits):
```bash
git filter-repo --path .env --invert-paths --force
```

**Explanation:**
- `--path .env` selects the file path to act on.
- `--invert-paths` means "remove this path from history (keep everything else)".
- `--force` skips interactive confirmation.
- After this runs, the `.env` will no longer exist in any commit. Commit SHAs will change for affected commits.

**Important follow-ups:**
- After rewriting history, you must force-push to remotes with care.
- All collaborators must reclone or reset their local clones after a history rewrite to avoid conflicts and accidental reintroduction of the removed file.
- Rotate any secrets that were previously leaked (treat them as compromised).

---

## 4️⃣ Untracking `.env`, Adding to `.gitignore`, and Committing Cleanup

After removing `.env` from history, ensure it is not tracked going forward and add it to `.gitignore`:

```bash
# Remove .env from Git index (unstage and stop tracking), keep the local file
git rm --cached .env

# Add .env to .gitignore to prevent future commits
echo ".env" >> .gitignore

# Commit the change (removed file is already absent from history via filter-repo)
git add .gitignore
git commit -m "Remove .env from repo and add to .gitignore"
```

**What this does:**
- `git rm --cached .env` removes the file from Git's index but leaves it on disk.
- `.gitignore` entry prevents accidental re-adds in future commits.
- Commit documents the cleanup and ignores rule.

---

## 5️⃣ Switching Remotes: GitHub → Hugging Face Space (HF)

When deploying to a **Hugging Face Space**, you will often point your local `origin` to the HF Space remote (or add `hf` as a separate remote).

**Check existing remotes:**
```bash
git remote -v
```

**Option A — Add new remote named `hf` (safe, keeps `origin` pointing to GitHub):**
```bash
git remote add hf https://huggingface.co/spaces/<your-username>/<your-space-name>
git remote -v  # verify 'hf' exists
```

**Option B — Replace `origin` URL to HF (common if you want origin to point to HF temporarily):**
```bash
git remote set-url origin https://huggingface.co/spaces/<your-username>/<your-space-name>
git remote -v  # verify origin now points to HF
```

**If you previously renamed remotes (e.g., `origin` → `github`):**
```bash
git remote rename origin github
# then you can set origin or add hf as needed
git remote add origin https://huggingface.co/spaces/<your-username>/<your-space-name>
```

**What this does:** Changes where `git push origin` will send commits. You can either keep multiple remotes (recommended) or temporarily point `origin` to HF while deploying.

---

## 6️⃣ Pushing to HF: Authentication Options & Examples

Hugging Face supports multiple authentication methods. Avoid embedding plain tokens in git URLs in shell history. Prefer `huggingface-cli login` or use CI secrets.

### Option 1 — Use HF CLI (recommended)
```bash
pip install huggingface-hub
huggingface-cli login  # follow prompt and enter HF token (stores creds locally)
# Then push normally (if origin points to HF):
git push origin main
# OR if you added 'hf' remote:
git push hf main
```

### Option 2 — Use Personal Access Token in the remote URL (less secure, avoid on shared machines)
```bash
# Example: push using token in URL (DO NOT paste tokens in shared logs)
git push https://<USERNAME>:<HF_TOKEN>@huggingface.co/spaces/<username>/<space-name> main --force
```

### Option 3 — Use remote name and token environment variable in CI
In CI, set `HF_TOKEN` as a secret and use it securely. Example (pseudo):
```bash
# in CI script: export HF_TOKEN=${{ secrets.HF_TOKEN }}
git remote set-url origin https://<USERNAME>:${HF_TOKEN}@huggingface.co/spaces/<username>/<space-name>
git push origin main --force
```

**Notes about force-pushing on first HF deployment:** HF Spaces may require an initial push that matches the expected structure. Using `--force` is common for the first push after history rewrites, but be cautious.

**Example commands you used previously (interpreted):**
```bash
git push https://huggingface.co/spaces/kaisarhossain/Smart-Email-Classification-App main --force
# or using token in URL (not recommended to expose token in shell history)
git push https://kaisarhossain:<HF_TOKEN>@huggingface.co/spaces/kaisarhossain/Smart-Email-Classification-App main
```

---

## 7️⃣ Reverting Remote Origin Back to GitHub (HF → GitHub)

Once deployment is done, you can point `origin` back to GitHub so you continue development with GitHub as the source of truth.

```bash
# Set origin to GitHub URL again
git remote set-url origin https://github.com/<your-username>/<repo-name>.git

# Push to GitHub (force only if you rewrote history intentionally)
git push origin main --force
```

**Alternative:** If you kept separate remotes (`github` and `hf`), simply push to whichever remote you want:
```bash
git push github main
git push hf main --force  # if needed for HF
```

---

## 8️⃣ Useful Verification & Safety Commands

- View remotes and URLs:
```bash
git remote -v
```

- Check current branch:
```bash
git branch --show-current
# or
git branch
```

- See concise commit history:
```bash
git log --oneline --graph --decorate -n 50
```

- Search commit history for references to `.env` (quick check):
```bash
git log --all --pretty=format:"%h %ad %s" --date=short | grep ".env" || echo "No .env references found in log output."
```

- Verify `.env` is untracked:
```bash
git ls-files --others --ignored --exclude-standard | grep ".env" || echo ".env not listed among ignored/untracked files."
```

- After `git filter-repo`, to ensure no `.env` blobs remain:
```bash
git rev-list --objects --all | grep ".env" || echo "No .env objects found."
```

---

## 9️⃣ Recommended Workflow Example (Complete)

A safe, end-to-end example for cleaning `.env`, pushing to HF, then reverting origin to GitHub:

```bash
# 0. Setup - ensure you are on the correct branch
git checkout main
git pull origin main

# 1. Install filter tool
pip install --upgrade git-filter-repo

# 2. Remove .env from entire history (destructive)
git filter-repo --path .env --invert-paths --force

# 3. Stop tracking .env and ignore it going forward
git rm --cached .env
echo ".env" >> .gitignore
git add .gitignore
git commit -m "Remove .env from repo and add to .gitignore"

# 4. Verify everything locally (optional checks above)

# 5. Add HF remote (safe approach: add new remote 'hf' and keep origin for github)
git remote add hf https://huggingface.co/spaces/<your-username>/<space-name>
git remote -v

# 6. Login to HF CLI (secure) and push
huggingface-cli login
git push hf main --force   # if history was rewritten, --force may be needed

# 7. Re-point origin back to GitHub (if you changed origin earlier)
git remote set-url origin https://github.com/<your-username>/<repo-name>.git
git push origin main --force
```

---

## 🔁 Appendix & Notes

- **Collaborators after a history rewrite:** They must re-clone the repository or run git commands to reset local history; otherwise they will get unpleasant merge conflicts or reintroduce removed files.
- **Alternative to `git filter-repo`:** BFG Repo-Cleaner is a simpler tool but less flexible for complex removals. `git-filter-repo` is recommended by Git project maintainers.
- **Store secrets securely:** Use environment variables in deployment platforms, or secret managers (GitHub Secrets, Hugging Face Secrets, AWS Secrets Manager, Google Secret Manager, etc.).

---

## 📬 Questions / Contact
If you want this README tailored with your HF and GitHub usernames pre-filled, I can create a version with the specific URLs and commands filled in. Also happy to add a short section about CI/CD (GitHub Actions) for automating HF pushes securely.
