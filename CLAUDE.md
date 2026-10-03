# CLAUDE.md

This repo is an AI resume builder that the founders cannot code themselves: Claude Code writes all code.

- Before any work, read `docs/build/CLOUD_BUILD_BRIEF.md` and `docs/build/BUILD_STATE.md`.
- The design spec is the source of truth: `docs/superpowers/specs/2026-10-03-resume-builder-design.md`. Use the names in its Appendix A exactly.
- Never commit secrets, never deploy, never use the name "Rezi", never weaken or skip tests.
- Milestone M0 expands this file with the full invariants from spec Appendix A section 8. Keep these first lines when you do.

## When the user types `start` (or `continue`)

Do not ask questions. Immediately:

1. Sync with the latest work: `git fetch origin`, then find the newest build branch with `git for-each-ref --sort=-committerdate --format='%(refname:short)' refs/remotes/origin/claude/` (prefer `origin/claude/mvp-build` if it exists and is newest). Check out `claude/mvp-build` if you can push to it; otherwise merge the newest build branch into your session branch.
2. Read `docs/build/CLOUD_BUILD_BRIEF.md` and `docs/build/BUILD_STATE.md` fully and follow the brief's hard rules and run protocol. In an interactive session you may skip the run lock and the 100-minute time box: keep working task after task until the session ends.
3. Order of work: SPEC (finish the design spec), then M0, M1, M2, M3, M4.
4. Commit and push after every task. If pushing to `claude/mvp-build` is refused, push to your own session branch and write its name on the `Branch:` line in `docs/build/BUILD_STATE.md`.
5. Before stopping, update `docs/build/BUILD_STATE.md` (ticked tasks, run log, Needs founder) and push.
