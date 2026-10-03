# Cloud build brief

You are a Claude Code cloud session building the MVP of an AI resume builder. You start with no memory of earlier runs; this file and `docs/build/BUILD_STATE.md` are your memory. Read both fully before doing anything.

## What you are building

- **Product:** an honest AI resume builder (feature-equivalent to market leaders, but an independent original product). Users build resumes, import existing ones, improve bullets with AI (with a check that flags any invented fact), tailor to a job description, and export PDF/DOCX. Sold as one-time passes in USD (Dodo Payments) and INR (Razorpay/UPI).
- **The design spec is the source of truth:** `docs/superpowers/specs/2026-10-03-resume-builder-design.md`. Appendix A (the canonical skeleton) fixes every folder, file, table, endpoint, page, module, id, script and secret name. Use those names exactly.
- **Architecture decision and verified facts:** `docs/decisions/0001-architecture.md`.
- **Product requirements and pricing:** `research/Simple_SaaS_Pick_AI_Resume_Builder.md`.
- **The team cannot code.** Code must be simple, consistent, heavily tested and safe by default. Nobody will review your code line by line; the tests and the invariants are the review.

## Hard rules (never break these)

1. Follow the 15 invariants in the spec (Appendix A section 8). They go into the repo-root `CLAUDE.md` in milestone M0.
2. **No secrets, no accounts, no deploys.** Never create real accounts, never request or invent API keys, never run `wrangler deploy` or touch remote D1, never commit `.dev.vars` or any secret. Use `.dev.vars.example` with placeholder names only.
3. **Build without external services:** `AI_PROVIDER=mock` for all AI, local D1 (Miniflare/wrangler local), Clerk verified with locally generated test JWT keys in tests, Dodo and Razorpay webhooks tested with locally signed fixtures. Anything that needs a real account is built behind its interface and listed under "Needs founder" in BUILD_STATE.md.
4. **Tests first** for every server, shared and billing module. `npm run typecheck`, `npm run lint`, `npm test` and `npm run test:int` must pass before every commit (once those scripts exist). Never skip, delete or weaken a test to make it pass. If a test needs a browser that cannot be installed here (Playwright golden tests), keep the test and record "run locally" in BUILD_STATE.md.
5. **Never use the name "Rezi"** or copy any competitor's code, templates, wording or design. The product name is `BRAND_NAME` in `src/shared/config/brand.ts` (placeholder until the founders choose).
6. **Do not edit the spec.** If you find a gap or contradiction, choose the safest reasonable option, implement it, and log it in `docs/build/SPEC_ISSUES.md` (file, problem, decision).
7. Pin dependency versions (exact versions in `package.json`, commit the lockfile). Prefer the libraries named in the spec.

## Git workflow

- Work only on the branch `claude/mvp-build`. At the start: `git fetch origin`, then check out `claude/mvp-build` (create it from `origin/main` if it does not exist) and `git pull --rebase`.
- Commit after every completed task with a clear message (`M1: add resumes repository with IDOR tests`). Push after every commit: `git push -u origin claude/mvp-build`.
- Never force-push, never rewrite history, never push to `main`.

## Run protocol (each run)

1. **Lock check.** Read the `Run lock` line in BUILD_STATE.md. If it shows a start time less than 150 minutes ago and is not cleared, another run is active: print "another run is active" and stop immediately without changes. Otherwise set `Run lock: started <UTC time from date -u>` and commit and push that change first.
2. **If BUILD_STATE.md says `Status: COMPLETE`, clear the lock and stop.**
3. **Plan.** If the current milestone has no task list yet, write `docs/superpowers/plans/<milestone>-plan.md`: break the milestone (from spec section 10 and the relevant spec sections) into small tasks of 15-60 minutes, each with files to touch, tests to write first, and a done-check. If the superpowers `writing-plans` skill is available, use it. Copy the task list into BUILD_STATE.md.
4. **Build.** Take the next unchecked task, write tests, implement, run the checks, commit, push, tick the box in BUILD_STATE.md with a one-line note. Repeat. Use subagents for independent tasks if available, but never let two agents edit the same files.
5. **Security pass.** After finishing any task that touches `src/server/data/**`, `src/server/core/route.ts`, `src/server/core/auth.ts` or `src/server/billing/**`, re-read the diff against invariants 2, 3, 4 and 6 before committing. If a code-review or security-review skill is available, run it on that diff.
6. **Time box.** Check `date -u` between tasks. After about 100 minutes of work, finish the current task, then stop starting new ones.
7. **Wrap up.** Update BUILD_STATE.md: tick finished tasks, update `Current milestone`, add a dated entry under `Run log` (what you did, test status, anything blocked), add any founder actions under `Needs founder`, then set `Run lock: cleared`. Commit and push.
8. When milestones M0 to M4 are all done and every check passes, set `Status: COMPLETE` and write a final summary at the top of the run log.

## Milestone SPEC (do this first)

The design spec was not finished locally. Only the canonical skeleton exists: `docs/superpowers/specs/parts/00-skeleton.md` (binding names), plus any section files already in `docs/superpowers/specs/parts/`. Before M0:

1. Write each missing section file in `docs/superpowers/specs/parts/`, using the skeleton names exactly. Sections: `01-overview.md` (product, journeys, scope, metrics), `02-architecture.md` (diagrams, route() chokepoint, config, environments, CLAUDE.md contents), `03-data-model.md` (tables detail, resume zod schema with limits and an Indian-fresher example, deletion/export), `04-api.md` (every endpoint: request, response, errors, quota, rate limit), `05-frontend.md` (pages, /app screens, editor store, autosave, preview, 6 templates, paywall UI), `06-ai.md` (each AI feature, prompts, honesty checker algorithm, JD match formula, quotas, mock provider), `07-payments.md` (Dodo and Razorpay end to end, passes, computeAccess, coupons, paywall), `08-security-privacy.md` (threat model, secrets, DPDP/GDPR, logging rules), `09-quality-ops.md` (tests, CI/CD, backups, runbooks for non-coders), `10-roadmap.md` (accounts in order, milestones M0-M6 with click-to-check acceptance criteria).
2. Inputs: the skeleton, `docs/decisions/0001-architecture.md` (verified facts) and `research/Simple_SaaS_Pick_AI_Resume_Builder.md` (requirements, pricing). Be precise; prefer tables and code-shaped specs; no TBD.
3. Assemble `docs/superpowers/specs/2026-10-03-resume-builder-design.md`: a short header and table of contents, sections 01-10, then "Appendix A: Canonical Skeleton" (the skeleton with headings demoted one level).
4. Review it once for security holes, Workers Free limits (10 ms CPU, 50 D1 queries per request), payment flow errors and contradictions; fix them in place. Commit and push after each section.

## Milestones for the cloud build

Build M0 to M4 from spec section 10 (Build Roadmap). M5 (closed beta) and M6 (launch) need the founders and real accounts; do not attempt them.

- **M0:** repo scaffold, toolchain, CLAUDE.md with the invariants, CI workflows, lint/boundary/brand/bundle/migration checks, empty Worker with `/api/health`, test harness.
- **M1:** auth integration (Clerk, test-key path), both D1 schemas and migrations, scoped repositories, resumes and versions API, editor with autosave, 2 templates, live preview and PDF export.
- **M2:** PDF/DOCX import, all AI features through the guarded LLM with the mock provider, honesty checker, JD keyword extraction and match %, tailored versions, all 6 templates, DOCX export.
- **M3:** plan catalog, Dodo and Razorpay flows with signed-fixture tests, passes ledger and computeAccess, quotas, coupons, the free no-signup checker with Turnstile verification behind an interface.
- **M4:** legal page templates (clearly marked as drafts for a lawyer/CA to review), privacy features (consent, export, deletion across both databases), analytics events, admin funnel and cost page, runbooks and `docs/accounts-setup.md` for the founders.

If M0 to M4 are complete and time remains: improve test coverage on critical invariants, then draft (not publish) up to 5 SEO articles from spec section 5 in the content collection, marked `draft: true`.

## Definition of done for the whole cloud build

- All M0-M4 tasks ticked in BUILD_STATE.md.
- `npm ci && npm run typecheck && npm run lint && npm run check && npm test && npm run test:int && npm run build` all pass on a clean checkout.
- `docs/accounts-setup.md` lists, in order, every account and secret the founders must create, and where each value goes.
- `Needs founder` in BUILD_STATE.md lists everything that could not be finished without humans.
