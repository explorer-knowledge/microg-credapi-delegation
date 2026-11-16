# CLAUDE.md

**Read `.docs/project.md` first.** It explains what this repo is and its hard constraints.

| Doc | Read when |
|-----|-----------|
| `.docs/project.md` | always (goal, layout, constraints, signing) |
| `.docs/implementation_plan.md` | changing the patch or the workflow (how-to, CI steps, status) |
| `.docs/issues.md` | before claiming something works; add new risks here |
| `.docs/audits_reviews.md` | before/after reviews; log findings and their status here |

## Rules
- **Never build, lint or test Android code locally** (no gradle, no SDK). The PC is too weak; GitHub Actions does it.
- **Only maintain `patches/`.** Don't modify, refactor or review upstream microG code outside the patch.
- Regenerate the patch from a source-only clone in `/tmp` and check it applies to the latest upstream release
  tag (steps in `implementation_plan.md`).
- Never commit keystores. The key lives at `~/.local/share/microg-credapi-delegation/debug.keystore`.
- `.docs/` is git-ignored on purpose; `README.md` is the public doc.
- Keep `.docs/` up to date after every change.
