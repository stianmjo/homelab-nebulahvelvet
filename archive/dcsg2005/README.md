# dcsg2005 — Overleaf Git Backup

Nightly mirror of the Overleaf project `69a818a34f812c729fd65773` to the NAS at
`10.0.2.5:/var/nfs/shared/media/dcsg2005-dcsg2005-backup-nfs/overleaf-dcsg2005`.

Runs daily at 21:00 Oslo time via a Kubernetes CronJob. Uses `git clone --mirror`
on first run, then `git remote update --prune` on subsequent runs — giving a full
bare clone of all commits, branches, and tags.

---

## Restore Options

### 1. Clone a working copy from the NAS mirror

The quickest way to get your files back locally:

```bash
git clone /path/to/nas/overleaf-dcsg2005 my-working-copy
cd my-working-copy
# Full history and all files available
```

### 2. Push the mirror to a new GitHub repo

Create an empty repo on GitHub first, then:

```bash
cd /path/to/nas/overleaf-dcsg2005

git remote set-url origin https://github.com/stianmjo/new-repo.git
git push --mirror
```

`--mirror` pushes all branches, tags, and full history — nothing is lost.

### 3. Restore to a new Overleaf project

Create a new blank project on Overleaf, grab its project ID, then:

```bash
cd /path/to/nas/overleaf-dcsg2005

git remote set-url origin https://git:<token>@git.overleaf.com/<new-project-id>
git push --mirror
```

### 4. Inspect history without restoring

Browse and extract individual files without a full restore:

```bash
# View full commit history
git --git-dir=/path/to/nas/overleaf-dcsg2005 log --oneline

# Extract a specific file from any commit
git --git-dir=/path/to/nas/overleaf-dcsg2005 show <commit-hash>:main.tex > main.tex

# See what changed between two points
git --git-dir=/path/to/nas/overleaf-dcsg2005 diff <hash-a>..<hash-b>
```

---

## Manually trigger a backup run

```bash
kubectl create job --from=cronjob/git-backup git-backup-manual -n dcsg2005
kubectl logs -f -n dcsg2005 -l job-name=git-backup-manual
```

---

## Secret

Credentials are stored in `git-backup-secret` in the `dcsg2005` namespace.
See `secret-git-token-template.yaml` for the expected shape.

```bash
# Rotate the token
kubectl delete secret git-backup-secret -n dcsg2005
kubectl create secret generic git-backup-secret \
  --namespace=dcsg2005 \
  --from-literal=OVERLEAF_PASS='<new-token>'
```