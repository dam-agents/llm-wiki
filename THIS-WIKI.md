# This wiki

_Not yet onboarded._

Run the `onboard` skill to specialise this wiki for its domain and sources.
Onboarding replaces the body of this file with:

- **Purpose** — what this wiki is for, in one line.
- **Domain vocabulary** — the terms this domain uses and what they mean here.
- **Entity vs concept** — the rule for what counts as an entity versus a concept
  in this domain (the taxonomy split is domain-specific).
- **Contradiction policy** — how to treat conflicting claims across sources for
  this domain: reconcile factual disagreements, or preserve genuine tensions.

`CLAUDE.md` is the shared operating manual and points here for everything
instance-specific. This file is **agent-owned runtime state**: it is
`skip-worktree`'d on the volume (installation does this), so onboarding's edits
never enter a definition commit, and it is never pushed to the wiki remote.
