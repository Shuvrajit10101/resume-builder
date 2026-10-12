# resume-builder

Planning and specification repo for an AI resume builder: a web app where a job seeker keeps one structured **master resume**, makes tailored versions of it for individual job descriptions, and exports ATS-readable PDF/DOCX with a real text layer. The AI rewrites the user's own wording and is checked for invented facts, with the diff shown before anything is accepted. It is sold as one-time passes (USD via Dodo Payments, INR via Razorpay/UPI) rather than a subscription.

**There is no application code here yet.** This repo currently holds the architecture decision, the design spec that is still being written, and the market research the product choice came from. The implementation is written by Claude Code sessions working from that spec; `CLAUDE.md` and `docs/build/` are their instructions and their state between runs.

## Structure

| Path | What it is |
| --- | --- |
| `CLAUDE.md` | Instructions for the Claude Code sessions that write the code |
| `docs/decisions/0001-architecture.md` | ADR 0001 — the chosen architecture, accepted 2026-10-03 |
| `docs/superpowers/specs/parts/` | Design spec sections. Written so far: `00-skeleton.md` (binding names for every folder, table, endpoint and env var), `01-overview.md`, `03-data-model.md` |
| `docs/build/CLOUD_BUILD_BRIEF.md` | Hard rules, git workflow and run protocol for a build session |
| `docs/build/BUILD_STATE.md` | Milestones M0–M6, the "needs founder" list and the run log |
| `research/` | Market research: an India SaaS opportunity atlas and the memo that picked this product |

## Status

Per `docs/build/BUILD_STATE.md`: `Status: IN PROGRESS`, current milestone `SPEC`, no milestone ticked. The assembled spec (`docs/superpowers/specs/2026-10-03-resume-builder-design.md`) does not exist yet — sections 02 and 04–10 are still to be written. Nothing is deployed, and the product name is still the placeholder `BRAND_NAME`.

## Planned stack

Decided in ADR 0001, not yet built: TypeScript throughout; Astro for the static marketing, pricing and legal pages; a React 19 single-page editor; Hono on one Cloudflare Worker for `/api`; Cloudflare D1 split into `core` and `docs` databases with Drizzle; Clerk for sign-in; OpenAI for the AI features behind a provider interface with a mock for local work; Dodo Payments and Razorpay for checkout. PDF/DOCX export, file import and the keyword-match maths run in the browser so the Worker stays inside the free tier's 10 ms CPU limit per request.

## Running it

Nothing to run yet — no `package.json` and no `src/`. The spec's section 1 defines the repository tree and npm scripts the build will create.
