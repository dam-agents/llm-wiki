# Installation — first-run machine setup

This runbook turns a fresh, generic Claude Code agent into the llm-wiki agent.
**Run it only once per agent instance**, at setup. You run it yourself, with your
own tools — there is no external installer. It is self-guarding: the first
successful pass writes a sentinel file, and every later run is a no-op.

Installation is **machine setup only**. It does not interview you, schedule
anything, or ingest any source — those belong to the `onboard` skill, which needs
answers installation cannot have yet (purpose, sources, the wiki remote, the
cadence). Installation ends by handing off to `onboard`.

Two git repositories exist after installation, and they must never overlap:

| Path | Repo | Purpose |
| --- | --- | --- |
| `/home/agent/work` | `dam-agents/llm-wiki` (`origin`) | The agent's **definition** (`CLAUDE.md`, `INSTALLATION.md`, `skills/`, `scripts/`). Edit + PR to evolve the agent. |
| `/home/agent/work/wiki` | the wiki `remote` (set at onboard) | The agent's **content** — the wiki pages. Created by `onboard`. |

`/home/agent/work` is Claude's working directory and is **below** `$HOME`
(`/home/agent`), so the agent's secrets and harness state (`.ssh`, `.claude`,
`.config`) sit outside this repo entirely. The allowlist `.gitignore` keeps
`sources/` and the inner `wiki/` content invisible to the outer repo.

## Guard — skip if already installed

```bash
if [ -f "$HOME/.llm-wiki-installed" ]; then
  echo "Already installed ($(cat "$HOME/.llm-wiki-installed")); skipping installation."
  exit 0
fi
```

If the sentinel exists, **stop here**. It lives in `$HOME`, which persists across
pod restarts, so installation runs once per persistent volume (once per real agent
instance), not once per session.

First make sure git auth is routed through `gh` (idempotent — same helper used for
this repo, for cloning sources, and for pushing the wiki):

```bash
git config --global --replace-all credential."https://github.com".helper "" \
  && git config --global --add credential."https://github.com".helper "!gh auth git-credential"
```

---

## Step 1 — Make `/home/agent/work` the llm-wiki repo

Establish the repo in place (do **not** `git clone` into the working dir — clone
needs an empty dir; instead init + fetch + hard-reset, which never touches
untracked files like an already-present `sources/` or `wiki/`):

```bash
REPO="dam-agents/llm-wiki"
cd "$HOME/work"

if [ ! -d "$HOME/work/.git" ]; then
  git init -q
  git remote add origin "https://github.com/$REPO.git"
fi
git fetch -q origin main
git reset --hard origin/main      # syncs tracked files (CLAUDE.md, skills/, scripts/, seeds) to canonical main
git branch --set-upstream-to=origin/main main 2>/dev/null || true
```

> **NEVER run `git clean` here** and never `git add` un-allowlisted paths — both
> could capture or delete `sources/`, the inner `wiki/`, or harness state.
> `git reset --hard` only rewrites *tracked* files and leaves untracked contents
> alone, which is why it is safe. The allowlist `.gitignore` keeps `git status` /
> `git add -A` scoped to the definition + instance files.

## Step 2 — Surface the skills

The four wiki skills (`onboard`, `ingest`, `query`, `lint`) ship in this repo
under `skills/`. The harness reads skills from `~/.agents/skills/` (surfaced to
Claude Code through the inherited `~/.claude/skills` symlink). Symlink each skill
from the repo into that tree so a later `git pull` of the definition keeps them
fresh automatically:

```bash
mkdir -p "$HOME/.agents/skills"
for d in "$HOME/work/skills"/*/; do
  name="$(basename "$d")"
  rm -rf "$HOME/.agents/skills/$name"
  ln -s "$HOME/work/skills/$name" "$HOME/.agents/skills/$name"
done
ls -la "$HOME/.agents/skills" | grep -c '\-> .*/work/skills/'   # expect 4
```

If symlinking is not possible on this volume, copy instead
(`cp -R "$HOME/work/skills/$name" "$HOME/.agents/skills/$name"`) and re-run this
step after any definition update.

## Step 3 — Protect the instance files

`wiki.config.json` and `THIS-WIKI.md` are tracked seeds (so a fresh checkout has
them), but on this volume they become per-instance runtime state that `onboard`
rewrites. Tell git to ignore those local modifications so they never leak into a
definition commit — without staging any deletion:

```bash
cd "$HOME/work"
git update-index --skip-worktree wiki.config.json THIS-WIKI.md
# The wiki/ seed becomes the inner content repo; ignore the outer repo's view of it too.
git update-index --skip-worktree wiki/index.md wiki/log.md 2>/dev/null || true
git status --porcelain
```

`git status` must show a **clean** outer tree — nothing under `sources/` or
`wiki/`, no instance-file changes. If anything there appears, the allowlist
`.gitignore` is wrong (or skip-worktree didn't take) — **stop and fix it before
continuing**; do not write the sentinel.

> Why `--skip-worktree` rather than `git rm --cached`: `rm --cached` would stage a
> deletion that a later definition commit could accidentally push (re-removing the
> seeds and breaking a fresh checkout). `--skip-worktree` leaves the index
> untouched and simply makes git ignore local edits to those tracked files.

## Step 4 — Write the sentinel

Only after Steps 1–3 succeeded:

```bash
date -u +%Y-%m-%dT%H:%M:%SZ > "$HOME/.llm-wiki-installed"
echo "Installation complete."
```

## Step 5 — Hand off to onboarding

Installation is machine setup; it does **not** schedule or ingest. The wiki is not
specialised yet (`THIS-WIKI.md` still reads _Not yet onboarded_, `wiki.config.json`
`purpose` is `null`).

Tell the user, in one line, that you are installed but not yet onboarded — you
build and maintain an AI-curated wiki that distils GitHub repos into cited,
interlinked markdown — and offer to set it up now. On a yes, run the **onboard**
skill, which runs the interview, writes the config and `THIS-WIKI.md`, schedules
recurring maintenance, and runs the first ingest.

From now on the guard at the top short-circuits, and normal runs follow
[`CLAUDE.md`](CLAUDE.md).
