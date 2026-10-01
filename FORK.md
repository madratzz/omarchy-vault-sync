# About this fork

This is a fork of [chyld/omarchy-vault-sync](https://github.com/chyld/omarchy-vault-sync).
It adds two things to the upstream plugin:

1. **Forgejo, Gitea and Codeberg support.** Any `https://<host>/<owner>/<repo>`
   URL works, not only GitHub.
2. **"Vault is the whole repository" mode.** It syncs a vault that is already
   a plain git clone with the notes at the top level, the way Obsidian Git or a
   hand-kept repository lays them out. It also uploads Git LFS files.

Everything else is unchanged: it never force-pushes, never resets, and keeps
both versions of a conflicted note. GitHub URLs in the default `Vaults/<name>/`
mode behave exactly as they do upstream, including the refs they already use.

```bash
omarchy plugin add https://github.com/madratzz/omarchy-vault-sync --enable
```

The plugin id is still `chyld.vault-sync`, so remove the upstream copy first
(`omarchy plugin remove chyld.vault-sync`). `~/.config/vault-sync/repos.json`
is kept across the swap.

> **After installing or updating, restart the shell** (`omarchy restart shell`).
> `Safe.js` and `Commands.js` are `.pragma library` scripts, which QML shares
> per engine. Omarchy hot-reloads the plugin's QML on update, but these scripts
> appeared to keep running the old code until a restart. The symptom is the
> popup still rejecting a non-GitHub URL with "Use a URL like
> https://github.com/…".

---

## 1. Forgejo, Gitea and other https hosts

### What changed

| Where | Before | Now |
|---|---|---|
| URL check (`Safe.repoUrl`, `engine.repo_url`) | only `https://github.com/<owner>/<repo>` | any `https://<host>[:port]/<owner>/<repo>`, host lowercased. `http://`, SSH, `user@` and IP addresses are still rejected. |
| Public-repo check (`Commands.visibility`) | `api.github.com/repos/<owner>/<repo>` | the same on GitHub, else `https://<host>/api/v1/repos/<owner>/<repo>` (Forgejo/Gitea). Same anonymous 200 = public, 404 = private rule. Any other status, such as the 403 from a server that requires sign-in, shows no warning. |
| Sync refs (`engine.sync_refs`) | `refs/vault-sync/<owner>/<repo>` | unchanged on GitHub, else `refs/vault-sync/<host>/<owner>/<repo>`. A `:` in a port becomes `_`. Repositories on different hosts never share state. |
| Wording | "GitHub" everywhere | Progress steps, errors, commit messages, the popup icon and the **Open on …** button name the host. The "login needed" error explains credential helpers instead of `gh`. |

### Signing in to Forgejo

Vault Sync never prompts for a password (`GIT_TERMINAL_PROMPT=0`,
`GIT_ASKPASS=/usr/bin/true`). git must already have a login for the host from
a credential helper. `gh auth setup-git` only covers GitHub, so for Forgejo:

1. **Create a token.** On your Forgejo server go to Settings → Applications →
   Generate new token, and give it **repository: Read and write**. If the
   repository belongs to an organization, your account also needs write access
   to it there.
2. **Store it.** Run this in your own terminal, so the token stays out of
   shell history and logs. It goes into your keyring through libsecret:

   ```bash
   git config --global credential.helper /usr/lib/git-core/git-credential-libsecret   # if not set already
   read -rp "Forgejo username: " u; read -rsp "Token: " t; echo
   printf 'protocol=https\nhost=git.example.com\nusername=%s\npassword=%s\n\n' "$u" "$t" | git credential approve; unset t
   ```

3. **Check it the way the plugin connects:**

   ```bash
   GIT_TERMINAL_PROMPT=0 GIT_ASKPASS=/usr/bin/true git ls-remote https://git.example.com/you/notes.git
   ```

   This must list refs without asking for anything. Git LFS uses the same
   credential helper.

---

## 2. "Vault is the whole repository" (root mode)

### Why it exists

Upstream's design is **one repository, many vaults, each in `Vaults/<name>/`**.
The plugin builds the pushed commit itself (`git commit-tree`). It takes the
remote tree, replaces `Vaults/<name>/` with the vault, and pushes that. The
vault's own branch is never pushed. That works when the plugin owns the
repository.

Point it at a repository where the vault **is** the repository, with notes at
the top level, and this happens:

1. Each sync pushes a commit that adds a second copy of the whole vault under
   `Vaults/<name>/`. Your local `main` and the remote `main` split apart: a git
   client shows ↑1 ↓1, with `Sync <date> — N files changed` (local) and
   `Sync Vaults/<name> <date>` (remote) on two lines.
2. A normal `git pull` merges that copy into the vault on disk. Obsidian then
   indexes every note twice.
3. The next sync copies the vault, copy included, into `Vaults/<name>/` again,
   so you get `Vaults/<name>/Vaults/<name>/…`. Every pull and sync nests it one
   level deeper.

Nothing is deleted, but the repository and the vault fill up with duplicates.

### What root mode does

Tick **Vault is the whole repository** under the URL in the popup. It is
stored per repository as `"root": true` in `~/.config/vault-sync/repos.json`
and passed to the engine as `--root`. Sync now then works on the vault's own
branch:

1. **Commit** everything git doesn't ignore (`git add -A -- :/`). Your
   `.gitignore` decides what is shared, `.obsidian` and `.trash` included.
   Files of 100 MB or more stop the sync, as upstream.
2. **Fetch** the remote branch with the same name as the vault's branch.
3. **Merge** it into the vault's branch. It fast-forwards when it can, else
   makes a merge commit, `Sync: merge changes from <host>`. A note changed on
   both sides keeps the remote version under its name and this machine's as
   `note (conflict …).md`, as upstream.
4. **Upload Git LFS files** with `git lfs push origin <branch>`, when the
   repository uses LFS (see below).
5. **Push** `HEAD` to the branch. It is never forced. If the remote moved on
   during the sync, it fetches and merges once more.

The result is an ordinary history: your commits, your branch, normal merges.
Status ("N changed", "N to send") counts the whole working tree and
`HEAD --not --remotes=origin`. A push from another git client counts as
synced.

A root repository holds **one vault**. Ticking another vault replaces it, and
the engine exits with an error if given more than one vault with `--root`.

### Guards: the two layouts can never be mixed

- **Folder mode refuses a clone.** In `Vaults/<name>/` mode, the vault's
  history never contains the remote's commits. If `git merge-base HEAD
  <remote tip>` finds shared history, the vault is a clone of a root-layout
  repository. The sync stops before it changes anything, with *"This vault is
  a clone of the repository with the notes at its root. Tick "Vault is the
  whole repository" to sync it."*
- **Root mode refuses unrelated histories.** It never passes
  `--allow-unrelated-histories`. A vault with notes of its own and a
  non-empty repository it doesn't share history with stops with *"The vault
  and the repository have no history in common."* Clone the repository into
  the vault folder first. An empty repository is fine: the first sync pushes
  the vault into it.

### Git LFS

git-lfs uploads files from a `pre-push` hook. The engine runs git with
`core.hooksPath=/dev/null`, so the hook never runs, and a plain push would
leave the server holding pointers to objects that were never uploaded. In root
mode the engine checks whether the repository uses LFS: either some tracked
files have `filter=lfs`, or `.gitattributes` mentions `filter=lfs`. If so, it
runs `git lfs push origin <branch>` before pushing. Merges that check out LFS
files download them through the normal smudge filter, with a longer deadline.
If the repository uses LFS but git-lfs isn't installed, the sync stops with
an explanation instead of pushing pointers.

Conflict copies now go through the path's filters (`git cat-file --filters`).
A conflicted LFS file is copied as its contents, not as its pointer. This
applies to both modes.

### Security trade-off

Upstream never lets `.obsidian` (settings and **plugin code**) or `.trash`
arrive from the remote. In root mode they do, exactly as with a plain
`git pull`, because the repository tracks them and filtering them out of
merges would delete them. Use root mode only with a repository you control,
and keep it private.

---

## Recovering a vault that got nested

This applies if you synced a root-layout vault in folder mode before these
guards existed, and your vault or repository now contains `Vaults/<name>/`
(possibly nested). Stop syncing and pulling, then in the vault:

```bash
cd /path/to/vault
git status                        # must be clean; commit or stash first
git fetch origin

# What differs outside Vaults/? Usually only your newest local edits.
git diff --stat HEAD origin/main -- . ':!Vaults'

# Join the histories, keeping this vault's top-level notes as they are...
git merge -s ours origin/main -m "Merge Vault Sync's pushes, keeping the vault at the repository root"
# ...then drop the copy, in git and on disk.
git rm -r -q Vaults && git commit -m "Remove the duplicate Vaults/ copy left by Vault Sync"
rm -rf Vaults

git push origin main              # a normal push: nothing is rewritten
git for-each-ref --format='%(refname)' refs/vault-sync | xargs -r -n1 git update-ref -d
rm -f .git/vault-sync-index
```

`-s ours` is only right when the `git diff --stat` above shows nothing you
want from the remote outside `Vaults/`. If it does, merge normally instead and
then remove `Vaults/`. Afterwards, tick **Vault is the whole repository** and
sync. The first sync records the state, and the vault shows as synced.

---

## Tests

```bash
node --test tests/
python3 -m unittest discover -s tests
```

The fork adds tests for:

- **URLs:** host URLs and ports, and rejected forms.
- **Public-repo check:** the Forgejo API URL it asks.
- **`--root` command lines:** including the one-vault limit.
- **The `root` flag in `repos.json`:** reading and writing it.
- **Root mode sync:** merging and pushing the vault's own branch, both versions
  kept on a conflict, unrelated histories refused, an empty repository, and LFS
  objects actually uploaded (skipped if git-lfs isn't installed).
- **The folder-mode guard:** a cloned vault is refused.

The engine tests use a `file://` remote behind `url.<base>.insteadOf`, and a
second `insteadOf` makes a Forgejo-style URL resolve to it.

## Keeping up with upstream

```bash
git remote add upstream https://github.com/chyld/omarchy-vault-sync   # once
git fetch upstream && git merge upstream/main
node --test tests/ && python3 -m unittest discover -s tests
git push origin main
omarchy plugin update chyld.vault-sync && omarchy restart shell
```
