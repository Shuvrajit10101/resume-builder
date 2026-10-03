## 1. Product Overview, Scope and User Journeys

*Part 01 of the design spec, written 2026-10-03. All names (paths, tables, endpoint keys, events, settings, error codes) come from Appendix A (Part 00, `parts/00-skeleton.md`). Appendix A wins on any conflict. This part adds product behaviour, decisions at the journey level, and success metrics. Calendar dates assume Day 1 = Monday 2026-10-05 (Part 10).*

### 1.1 Product summary

| Item | Decision |
|---|---|
| What it is | A web app where a job seeker builds one structured **master resume**, makes honest **tailored versions** of it for individual job descriptions (JDs), and exports ATS-readable PDF/DOCX files with a real text layer. AI rewrites the user's own wording and never adds facts. |
| Name | `BRAND_NAME` from `src/shared/config/brand.ts`, written as `{BRAND_NAME}` in copy. It stays the placeholder `[BRAND]` until the founders pick a cleared name (Part 10, step A0). |
| Markets | Global: English, USD, Dodo Payments as merchant of record. India: INR through Razorpay, UPI first. One product and one codebase; currency is a per-user setting (`users.currency`). |
| Business model | A free tier with no time limit, plus one-time **passes** with an end date and no auto-renew (7/30/90 days), plus a "founder" lifetime plan that has monthly fair-use limits and is capped at `lifetime_founder_limit` sales. Campus cohorts get passes through 100%-off coupons. Plans and prices: Appendix A §6.1. |
| Lead magnet | A free checker at `/resume-checker` that needs no sign-up. It always shows a deterministic report, and it adds AI tips for up to `anon_checker_ip_daily_limit` requests per IP per UTC day. |
| Differentiators | (1) Honest tailoring with visible diffs and honesty flags (1.3). (2) A keyword match formula that is published at `/how-match-score-works`. (3) One-page India fresher/campus formats: CGPA or percentage, Class X/XII with board, projects, internships, positions of responsibility. (4) You pay per job search, not per month. |
| Constraints | $0/month until revenue. Claude Code writes 100% of the code. The site is static-first on Cloudflare (Appendix A §2.1). |

#### 1.1.1 Core loop

```
/resume-checker (anonymous) ──► sign up ──► master resume (blank | sample | import)
                                               │
                                               ├─► AI improve (diff + honesty flags, accepted one suggestion at a time)
                                               ├─► tailor to a JD ──► tailored version (+ match %) ──► cover letter (paid)
                                               └─► export PDF / DOCX
                         any limit reached ──► paywall ──► pass (USD via Dodo | INR via Razorpay | coupon)
```

#### 1.1.2 What each tier gets (summary for copy)

The exact numbers are in Appendix A §6.3. This table is the plain-language summary used on `/pricing`, in `PaywallDialog` and in the FAQ.

| Capability | Free | Any time-limited pass | Lifetime (founder) |
|---|---|---|---|
| Editing, live preview, content checks, checker | Unlimited | Unlimited | Unlimited |
| Master resumes / tailored versions kept | 2 / 1 | 10-20 / 20-100 (depends on the pass) | 20 / 100 |
| AI actions (bullet rewrite, summary, cover letter) | 5 per 30-day free period | 60-600 per pass | 150 per UTC month |
| AI import / JD analysis | 1 / 1 per 30-day free period | 5-20 / 20-150 per pass | 10 / 50 per month |
| PDF export | 1 per 30-day free period | 100-900 per pass | 300 per month |
| DOCX export, cover letters | Not included | Included | Included |
| AI writing quality for bullet/summary | Standard model | Higher-quality model | Higher-quality model |

Copy never names model ids. Those strings exist only in `llm-config.ts` (Appendix A invariant 5, `check:boundaries`).

**Rule for published limits:** the static `/pricing` page and the FAQ render `DEFAULT_PLAN_LIMITS` at build time. Use `npm run set-limit` for emergency changes. Within 7 days, either revert the change or merge a PR that updates `DEFAULT_PLAN_LIMITS`, so that the published copy matches what is enforced. Never lower a paid limit group (`pass_*`, `lifetime`) below its published number while passes bought under that number are still active. Free-tier limits may be lowered temporarily to stop abuse.

#### 1.1.3 Product copy rules (apply to pages, emails from providers, templates, ads and posts)

| # | Rule |
|---|---|
| C1 | The product name is only ever `{BRAND_NAME}`. Never name a competitor in product copy, templates, ads, code or tests (Appendix A invariant 10). Comparison claims appear only in founder-written posts, and they must be factual and dated. |
| C2 | Never promise "beat the ATS", "guaranteed interviews/shortlists" or an "ATS score". Say "keyword match" and link to `/how-match-score-works`. |
| C3 | INR prices are shown GST-inclusive ("₹199"). USD prices are shown as "$9", followed by "Sales tax or VAT may be added at checkout depending on your country" (Dodo is the merchant of record). |
| C4 | Every pass is described as "one-time payment, no auto-renew, ends on <date>". The date is shown before payment as well (stacking: Appendix A §6.4). |
| C5 | Lifetime is "Founder lifetime: limited to the first {lifetime_founder_limit} buyers", with the monthly fair-use limits written out. |
| C6 | No fake testimonials, user counters, countdown timers or invented scarcity. Testimonials are published only with written consent. |
| C7 | AI copy follows "AI suggests, you decide." Never say "AI writes your resume for you". |
| C8 | Resume content is never used in marketing unless it is anonymised and the owner has given written consent (Part 10, manual track). |

### 1.2 Who it is for

#### 1.2.1 Personas

| Id | Persona | Context | Main need | Default template | Currency → plan they most likely buy | Channel |
|---|---|---|---|---|---|---|
| P1 | **Global job seeker**: early to mid-career, English-speaking (US, UK, CA, AU, EU, Gulf) | Applying to many roles; often already has a resume | Tailor quickly per job without inventing anything; clean ATS-readable export | `global_classic` | USD → `usd_30d` | Checker, guides, Reddit/LinkedIn answers, directories |
| P2 | **Indian fresher**: final-year student or graduate of 0-2 years, engineering/BCA/MBA/commerce | Campus placements and off-campus drives; on mobile, often inside WhatsApp/Instagram browsers | One-page fresher format with CGPA, projects and internships; affordable UPI pass | `india_fresher` / `india_tech_fresher` | INR → `inr_7d` or `inr_30d` | WhatsApp/Telegram groups, LinkedIn teardowns, checker |
| P3 | **Campus cohort student** | Their placement cell gave them a link | Get started fast; a free pass through the cohort | `india_campus_academic` | INR → 100%-off coupon (`couponRedeem`) | Placement-cell group link (J8) |
| P4 | **Placement-cell coordinator** (buyer, not a user in the MVP) | Runs a batch's placement prep | A pilot for one batch, then a cohort price | n/a | Pays the founders outside the product | In-person visit, email |
| P5 | **Founder/operator** | Runs the business without coding | See the funnel, costs and limits; switch AI or checkout off | n/a | n/a | `/app/admin`, `npm run` ops scripts |

#### 1.2.2 Not for (MVP)

Recruiters and hiring teams, career coaches managing many clients, enterprises that need seats/SSO, and anyone wanting a non-English UI.

#### 1.2.3 Context assumptions

| Topic | Decision |
|---|---|
| Devices | Editing is designed for desktop/laptop first. The checker, sign-in, purchase and export must work fully on a 360 px wide phone. On widths below 768 px the editor shows "Edit" and "Preview" as tabs instead of side by side. |
| In-app browsers | Google sign-in can be blocked inside WhatsApp, Instagram, LinkedIn or Telegram browsers. The email code option is always on the same screen (Clerk `<SignIn>` / `<SignUp>`). |
| Language | The UI is English only. Resume locales are `en-IN`, `en-US` and `en-GB`. |
| Connectivity | Indian mobile networks: marketing pages are static and hydrate only their islands. The editor saves a local draft so a dropped connection loses nothing. |

#### 1.2.4 Defaults by currency and country (new resumes)

`defaultsForUser(currency, country)` (SKELETON ADDITIONS, 1.9) returns the defaults for each new resume:

| Condition (evaluated in order) | `templateId` | `locale` | `pageSize` | Sample used by "Start from sample" |
|---|---|---|---|---|
| `currency = 'INR'` | `india_fresher` | `en-IN` | `A4` | `SAMPLE_RESUME_INDIA` |
| `currency = 'USD'` and `country = 'US'` | `global_classic` | `en-US` | `LETTER` | `SAMPLE_RESUME_GLOBAL` |
| otherwise | `global_classic` | `en-GB` | `A4` | `SAMPLE_RESUME_GLOBAL` |

`users.currency` itself defaults to `'INR'` when `country = 'IN'`, otherwise `'USD'` (Appendix A §2.2). The user can change it in onboarding, `/app/billing` or `/app/settings` (`meUpdate`).

### 1.3 The honest-tailoring promise

#### 1.3.1 Promise (canonical copy, used on `/`, `/pricing` and in onboarding)

> {BRAND_NAME} only rewrites what you actually did. Every AI suggestion shows exactly what changed. Anything that isn't already in your resume (a number, an employer, a degree, a skill or a date) is highlighted before you accept it.

#### 1.3.2 Rules

| # | Rule | Enforced by | Verified by |
|---|---|---|---|
| H1 | AI only rephrases content from the **honesty source**: the stored document (Appendix A §7.3) plus user-authored input. JD text and JD keywords are **never** part of the honesty source (1.10, issue 2). | `ai/tasks/*` prompts + `postProcess` with `buildHonestySource` | `honesty.test.ts`, `ai.test.ts` |
| H2 | Every AI output is a suggestion. Nothing changes the document until the user clicks **Accept** on that one suggestion. Discarding leaves the document untouched. | `AiSuggestPanel`, `ai-store.ts` | `editor-flow.spec.ts` |
| H3 | Each suggestion is shown as a side-by-side diff against the current text. | `DiffView` (jsdiff) | `editor-flow.spec.ts` |
| H4 | The server runs `findUnsupportedClaims` on every AI output before returning it. Flags of kind `number`, `percent`, `money`, `organization`, `degree`, `skill` or `date` are returned with offsets and highlighted. | `ai/guard.ts` step 5 + task `postProcess` | `ai.test.ts` |
| H5 | **Accept** stays disabled while any flag is `open`. The user resolves each flag by editing the suggestion until the flag disappears (flags are recomputed in the browser with the same `findUnsupportedClaims`) or by ticking "Keep anyway (this is true)". | `ai-store.ts` state machine (1.3.3) | `editor-flow.spec.ts` |
| H6 | Missing JD keywords are listed, never inserted automatically. When a bullet rewrite uses JD keywords, any keyword that is not in the honesty source comes back as a `skill` flag. | `MatchPanel`, H1 + H4 | `ai.test.ts` (fixture: JD keyword injected → flagged) |
| H7 | The AI import (`aiImport`) is checked against the extracted text. Every section is confirmed by the user before `resumeCreate`. | `ImportPage`, `flagResumeDraft` | `ai.test.ts`, `editor-flow.spec.ts` |
| H8 | The match % is deterministic, explained publicly, and never called an "ATS score". | `keywords/match.ts`, `/how-match-score-works` | `keywords.test.ts` |
| H9 | AI processing needs a separate `ai_processing` consent that can be withdrawn (anonymous checker: `aiConsent: true` on every request). OpenAI is called with `store:false`, and API data is not used for training. | `ai/guard.ts` step 1, `ai/openai.ts` | `ai.test.ts` (`CONSENT_REQUIRED`) |

#### 1.3.3 Suggestion state (`src/app/state/ai-store.ts`)

```ts
type FlagState = 'open' | 'kept';                       // a flag that disappears after an edit is simply removed
type Suggestion = {
  id: string;                                           // crypto.randomUUID()
  target: FieldRef | { coverLetter: true };
  original: string;
  text: string;                                         // the current candidate text, editable
  flags: HonestyFlag[];                                 // recomputed after every edit
  flagState: Record<string, FlagState>;                 // key = `${kind}:${text}` so the state survives offset shifts
};
const canAccept = (s: Suggestion) => s.flags.every(f => s.flagState[`${f.kind}:${f.text}`] === 'kept');
```

Kept flags are held only in memory. They are not sent to the server or logged, and no event records them.

### 1.4 Key user journeys

Format: steps (actor → system → route/endpoint), events, failure paths, and **Done when** (Given/When/Then statements that the E2E tests and the Part 10 acceptance clicks check). The UI response to each error code is defined once in 1.5.

#### J1. Anonymous checker → sign-up

| # | User | System | Route / endpoint |
|---|---|---|---|
| 1 | Opens the checker from a post, guide or search | Static page; `CheckerApp` mounts; first touch is captured (`captureFirstTouch`, see 1.10 issue 1) | `/resume-checker` |
| 2 | Drops a PDF/DOCX or pastes text; optionally pastes a JD | `extractText(file)` runs in the browser (20,000-char cap, scanned-PDF detection). No file is uploaded. | — |
| 3 | Optionally ticks "Also get AI tips" | Shows the AI notice (sent to OpenAI for this request only, not stored by us, not used for training), the `aiConsent` checkbox and a Turnstile widget | — |
| 4 | Clicks "Check my resume" | The deterministic report renders at once in the browser. With a JD, it adds a heuristic keyword match (`extractKeywordsHeuristic` + `computeMatch`). | — |
| 5 | (AI ticked) | `checkerTips` with text, optional JD, Turnstile token and `aiConsent: true`. Shows the top 5 fixes (+ AI JD keywords) under the report. | `POST /api/checker/tips` |
| 6 | — | Once step 5 settles (or right after step 4 when AI is not ticked): `trackPublic('checker_used', {fileKind, hasJd, aiTips})`. `aiTips` is `true` only if tips were displayed. Exactly one event per check. | `POST /api/events` |
| 7 | Clicks "Fix these in the editor" | `checker_cta_clicked {cta:'fix_in_editor'}`, then goes to `/app/sign-up?redirect=%2Fapp%2Fimport` ("See plans" sends `{cta:'pricing'}` → `/pricing`) | `POST /api/events` |
| 8 | Signs up with Google, email code or email link | Clerk `<SignUp routing="hash">` | `/app/sign-up` |
| 9 | Completes onboarding | `meInit` (creates the row, first-touch, campus; `signup_completed` on creation) → terms + privacy consent (required) → AI consent (optional, explained) → currency | `/app/onboarding?next=%2Fapp%2Fimport`, `meInit`, `meConsent` ×2-3, `meUpdate` |
| 10 | Lands on the import page and uploads the file again (no carry-over in the MVP, 1.6.2) | J3 | `/app/import` |

All `redirect` / `next` values pass through `safeRedirectPath` (1.9). Anything that is not an `APP_ROUTES` path falls back to `/app`.

**Failure paths:** `TURNSTILE_FAILED` (widget reset, retry); `RATE_LIMITED` (report stays, sign-up CTA); `AI_DISABLED` / `AI_BUDGET_EXHAUSTED` (report stays, "AI tips are paused"); `AI_PROVIDER_ERROR` / `AI_OUTPUT_INVALID` (one "Try again"); scanned PDF ("paste the text instead"); text over 20,000 characters (truncated, with a notice).

**Done when:**
- Given a logged-out visitor with a text PDF, when they click "Check my resume" without AI, then the report renders with no network call except `POST /api/events`.
- Given AI ticked and Turnstile solved, then up to 5 tips are shown, and with a JD the matched/missing keywords are shown.
- Given 3 AI checks from one IP on one UTC day, when a 4th is requested, then it fails with `RATE_LIMITED` and the deterministic report still renders.

#### J2. Build a resume from scratch

| # | User | System | Route / endpoint |
|---|---|---|---|
| 1 | On the dashboard clicks "New resume" and picks "Blank" or "Start from sample" | Count-cap check against `Me` first. At `resumes_max`, the paywall opens (trigger `resume_cap`). | `/app` |
| 2 | — | `resumeCreate`: `source:'blank'` with `emptyResume(templateId)` + `defaultsForUser` settings, title "Untitled resume"; or `source:'sample'` with the sample from 1.2.4, title "Sample resume". The server emits `resume_created`. | `POST /api/resumes` |
| 3 | — | Opens the editor | `/app/editor?resume=<id>` |
| 4 | Fills in section forms; drags sections (`SectionOrder`); hides or renames sections | `editor-store` updates; the PDF preview re-renders; `ChecksPanel` runs `runContentChecks` (with `pageCount` from `countPages`) | — |
| 5 | Stops typing / leaves the field / switches tab | `useAutosave` saves after 15 s idle, on blur and on `visibilitychange=hidden`, skipping the save when the content hash is unchanged. The IndexedDB draft (`putDraft`) covers the time until the server save succeeds. | `PUT /api/resumes/:id` (`baseRevision`) |
| 6 | Changes the template | `TemplatePicker` sets `settings.templateId` and keeps the user's `sectionOrder`. "Use recommended order" applies that template's default order (Appendix A §7.2). | — |
| 7 | Exports | J7 | — |

Save indicator states: `Saved` · `Saving…` · `Offline: saved on this device` · `Maintenance: saved on this device` (on `READ_ONLY`). Local drafts retry every 30 s and on `online`.

**Failure paths:** `REVISION_CONFLICT` → dialog "This resume was changed in another tab or device" with **Load saved version** (discards the local draft) or **Keep my version** (re-saves with `details.currentRevision` as `baseRevision`). `PAYLOAD_TOO_LARGE` → "This resume is over the 64 KB limit; remove some content". `PAYWALL` on create → paywall `resume_cap`.

**Done when:**
- Given a new free user, when they create a blank resume, type their name and the indicator shows `Saved`, then reloading shows the name.
- Given the same resume open in two tabs, when both are edited and saved, then the second save shows the conflict dialog.
- Given 2 existing masters on the free tier, when "New resume" is clicked, then the paywall opens and no request to create a third is sent.

#### J3. Import an existing resume

| # | User | System | Route / endpoint |
|---|---|---|---|
| 1 | Clicks "Import resume" | Count-cap check first (at `resumes_max` → paywall `resume_cap`, so the import quota is not spent) | `/app/import` |
| 2 | Uploads a PDF/DOCX or pastes text | `extractText` in the browser. Scanned PDF → "This PDF is an image; paste the text instead". The raw file never leaves the browser. | — |
| 3 | Clicks "Structure with AI" | If `ai_processing` is not granted, an inline consent card appears (`meConsent`) | `POST /api/me/consents` |
| 4 | — | `aiImport` consumes 1 `ai_import` and returns a `ResumeData` draft, per-section confidence, and flags from `flagResumeDraft` against the extracted text. The server emits `import_used`. | `POST /api/ai/import` |
| 5 | Reviews one card per non-empty section: **Accept**, **Edit** or **Discard**. Low-confidence sections start expanded, and flagged values are highlighted. | Draft held in memory only | — |
| 6 | Clicks "Create resume" (enabled once every card is accepted or discarded) | `resumeCreate` with `source:'import'`, the confirmed data and title "Imported resume" → `resume_created` → editor | `POST /api/resumes` |

**Failure paths:** no import quota (`PAYWALL` / `QUOTA_EXHAUSTED`), or `AI_DISABLED` / `AI_BUDGET_EXHAUSTED` → "Continue without AI" keeps the extracted text on screen with a **Copy** button and offers **Create blank resume**. `AI_PROVIDER_ERROR` / `AI_OUTPUT_INVALID` → retry (the quota was refunded).

**Done when:**
- Given a free user with an unused import and a text PDF fixture, when they import and accept all cards, then the editor opens with the content, and `QuotaMeter` shows the import as used.
- Given an image-only PDF, then no AI call is made and the quota is unchanged.

#### J4. Improve with AI (diff + honesty flags)

| # | User | System | Route / endpoint |
|---|---|---|---|
| 1 | Focuses a bullet and clicks "Improve with AI" (or "Write summary" in the summary section) | Consent check (card if missing). Any pending save is flushed and awaited, so the stored copy is current (Appendix A §7.3). | `PUT /api/resumes/:id` or `/api/tailored/:id` |
| 2 | Optionally ticks "Use job keywords" (only on tailored documents that have cached keywords) | — | — |
| 3 | — | `aiBullet` / `aiSummary` consume 1 `ai_action` and return up to 3 candidates, each with `HonestyFlag[]`. The server emits `ai_used`. | `POST /api/ai/bullet`, `POST /api/ai/summary` |
| 4 | Reviews `AiSuggestPanel`: `DiffView` (original vs candidate) and `HonestyFlagList` ("Not in your resume: '35%'") | — | — |
| 5 | Resolves each flag: edits the text, or ticks "Keep anyway (this is true)" | Flags are recomputed in the browser after each edit (1.3.3) | — |
| 6 | Clicks **Accept** (enabled only by `canAccept`) or **Discard** | Accept replaces the text in `editor-store` and pushes an undo entry (Undo button and Ctrl/Cmd+Z), then autosave runs. Discard changes nothing. Either way the action counts against the quota. | — |

**Failure paths:** `QUOTA_EXHAUSTED` (free tier → paywall `ai_quota`; pass or lifetime → notice "Limit reached; resets on <periodEndsAt>"); `CONSENT_REQUIRED` → consent card; `AI_DISABLED`; `AI_BUDGET_EXHAUSTED` ("AI is busy today; try again after 05:30 IST / 00:00 UTC"); `AI_PROVIDER_ERROR` / `AI_OUTPUT_INVALID` ("That didn't work. Your AI quota was not used.").

**Done when:**
- Given a free user, when 5 AI actions have been used, then the 6th shows the paywall with the free period's end date, and `QuotaMeter` shows 5/5.
- Given a candidate containing a number that is not in the document (mock-provider fixture), then the number is highlighted and Accept stays disabled until it is edited out or kept.
- Given Accept, when the page is reloaded after `Saved`, then the new text persists, and Undo before reload restores the old text.

#### J5. Tailor to a job description (and cover letter)

| # | User | System | Route / endpoint |
|---|---|---|---|
| 1 | On a master on the dashboard, clicks "Tailor for a job" | Count-cap check (`tailored_max` → paywall `tailored_cap`) | `/app/tailor/new?resume=<id>` |
| 2 | Pastes the JD (1-15,000 chars), job title (0-120) and company (0-120) | Client-side length validation | — |
| 3 | Clicks "Create tailored version" | `tailoredCreate` copies the master's data. Default title: `"<jobTitle> – <company>"` trimmed to 80 characters, or "Tailored resume" when both are empty. The server emits `tailored_created`. | `POST /api/tailored` |
| 4 | — | `aiJdKeywords` for the new id (1 `jd_analysis`; the cache sits on the row, and cache hits are free) | `POST /api/ai/jd-keywords` |
| 5 | — | The editor opens with `MatchPanel`: percent, matched/missing required and preferred keywords, and a "How is this calculated?" link | `/app/editor?tailored=<id>` |
| 6 | Edits the tailored copy (the master is never changed); uses J4 with "Use job keywords" | — | — |
| 7 | — | `tailoredSave`: the server recomputes `match_percent`, which the dashboard shows under the master | `PUT /api/tailored/:id` |

- **JD edited later** (`tailoredUpdateMeta` with `jdText`): `MatchPanel` shows "Job description changed. Re-analyse (uses 1 JD analysis)".
- **No AI keywords** (quota used up, AI off, provider error): `MatchPanel` uses `extractKeywordsHeuristic(jd_text)` in the browser, labelled "Basic keywords (not AI)". Nothing is cached, and the server-side `match_percent` stays `null`.

**J5b. Cover letter (paid):**
- `/app/cover-letter?tailored=<id>` → "Generate" → `aiCoverLetter` (1 `ai_action`, mini) → draft with flags → resolve flags (1.3.3) → Accept → `coverLetterCreate` (cap `cover_letters_max`); the URL gains `&id=<coverLetterId>`.
- Later edits use `coverLetterSave` with `baseRevision`. Exports go through J7, with the layout matching the tailored document's template family.
- On the free tier the page shows any existing letters, and "Generate" opens the paywall (trigger `cover_letter`).
- After a pass ends, existing letters stay readable, editable and deletable.

**Done when:**
- Given a JD fixture, then the match % and the keyword lists appear. When a missing required keyword is added truthfully to Skills and saved, the % goes up, and the dashboard shows the new value.
- Given `ai_enabled=false`, then `MatchPanel` shows basic keywords and the label.
- Given a pass holder, then a cover letter can be generated, accepted, saved, reloaded and exported.

#### J6. Paywall and purchase (USD or INR)

**Paywall triggers.** The client picks the `paywall_viewed` trigger from the **UI action** that hit the limit, so it does not depend on `details.reason` (1.10, issue 4). The dialog opens only when `access.tier === 'free'`. Pass and lifetime users get the "resets on <date>" notice instead.

| UI action | Error | `trigger` |
|---|---|---|
| AI bullet / summary | `QUOTA_EXHAUSTED` | `ai_quota` |
| AI import | `QUOTA_EXHAUSTED` or `PAYWALL` | `import_quota` |
| JD analysis | `QUOTA_EXHAUSTED` or `PAYWALL` | `jd_quota` |
| PDF export | `QUOTA_EXHAUSTED` | `pdf_limit` |
| DOCX export | `PAYWALL` | `docx` |
| Generate or save a cover letter | `PAYWALL` | `cover_letter` |
| New / import / duplicate master at the cap | `PAYWALL` (or the client-side cap check) | `resume_cap` |
| New tailored version at the cap | `PAYWALL` (or the client-side cap check) | `tailored_cap` |
| "Upgrade" in `AppHeader` or the dashboard | — | `nav` |

`PaywallDialog` → `trackUser('paywall_viewed', {trigger})` → "See plans" → `/app/billing?plan=<suggested>`. Suggested plan: `inr_30d` for INR, `usd_30d` for USD.

**Billing page rules:**
- It shows `plansForCurrency(Me.currency)` with a currency switch (`meUpdate`).
- Lifetime is hidden when `meta.lifetimeAvailable` is false (with a "sold out" note) or when `access.hasLifetime` is true.
- With an active pass, each card shows "Starts after your current pass ends on <date>".
- If `checkout_<currency>_enabled` is false, the buy buttons become "Payments open soon"; for INR the coupon field stays usable.

**USD flow (Dodo):**

| # | User | System | Route / endpoint |
|---|---|---|---|
| 1 | Clicks "Buy" on a USD plan | `checkout` creates an intent from `PLANS`; emits `checkout_started`; returns `checkoutUrl` | `POST /api/billing/checkout` |
| 2 | Pays on Dodo's hosted page (can enter a Dodo discount code there) | Dodo charges and adds tax; webhook → `grantPass` | `POST /api/webhooks/dodo` |
| 3 | Lands back on the return page | Polls `intentGet` every 2 s for up to 60 s. `paid` → "Pass active until <date>"; `failed` → "Payment failed" + retry; still `created` → "Payment received? Activation can take a few minutes. If it isn't active within 10 minutes, email support@<domain> with this reference: <intentId>" | `/app/billing/return?intent=<id>` |

**INR flow (Razorpay):**

| # | User | System | Route / endpoint |
|---|---|---|---|
| 1 | Optionally enters a coupon → `couponPreview` shows the discounted price | — | `POST /api/billing/coupons/preview` |
| 2 | Clicks "Pay ₹<amount>" | `checkout` (+ `couponCode`) → `{orderId, keyId, amountMinor, currency, intentId}`, emits `checkout_started` | `POST /api/billing/checkout` |
| 3 | Pays in the Razorpay modal (UPI/card) | The modal handler calls `razorpayVerify` → `grantPass`. The webhook grants the same pass idempotently, whichever arrives first. | `POST /api/billing/razorpay/verify`, `POST /api/webhooks/razorpay` |
| 4 | — | `Me` is refetched → success state "Pass active until <date>" → "Go to dashboard" | `/app/billing` |

**Failure paths:**
- Modal closed or payment failed: no pass. The intent stays `created` and expires after 48 h (`runDaily`); every new click creates a new intent.
- `CHECKOUT_DISABLED`, or `PLAN_UNAVAILABLE` (lifetime sold out, or plan/currency mismatch): handled in the UI as in 1.5.
- Refund or dispute: the founder issues refunds from the provider dashboard. The webhook → `revokePass` → `pass_revoked`, and the passes list shows "Revoked (refund)".

**Done when:**
- Given test mode, then a USD test purchase shows the pass as active on the return page within 60 s, and `QuotaMeter` shows the pass limits.
- Given test mode, then an INR test UPI payment shows the pass immediately. A refund from the provider dashboard returns the user to the free tier within 5 minutes.

#### J7. Export PDF / DOCX

| # | User | System | Route / endpoint |
|---|---|---|---|
| 1 | Clicks "Export" | `ExportDialog`: format (PDF/DOCX), current template, page count (`countPages`) | — |
| 2 | — | On an India (one-page) template with more than 1 page: a warning that does not block ("This template is designed for one page; yours has N pages"), linking to the `page_overflow` check | — |
| 3 | Clicks "Download" | Flushes any pending save, then `exportAuthorize({docKind, docId, format, templateId})` consumes `pdf_export` / `docx_export` and writes `export_used` (and `first_export` the first time) | `POST /api/exports` |
| 4 | — | Renders in the browser (`renderResumePdf` / `renderResumeDocx` / cover letter variants) → `saveBlob(blob, exportFileName(basics, kind, format))` | — |

The export gate is soft by design: rendering happens in the browser, so a determined user could get around it. This is an accepted trade-off for $0 hosting (Appendix A §3).

**Failure paths:**
- Free tier: a second PDF in the period → `QUOTA_EXHAUSTED` → paywall `pdf_limit`. DOCX → `PAYWALL` → `docx`.
- A render error shows "Export failed; try again" and reports to Sentry without content (render-order issue: 1.10, issue 5).

**Done when:**
- Given a free user, then the first PDF downloads, its text can be selected, and it copies in reading order with `₹` intact. The second PDF opens the paywall.
- Given a pass holder, then DOCX downloads and opens in Word/Google Docs with the sections in order.

#### J8. Campus coupon redemption

**Founder side (no code):**
1. Agree a free one-batch, one-season pilot with a placement cell.
2. Choose a campus code (uppercase, for example `ABCENGG`) and a coupon code. Run `npm run coupon:create` with `--env production`, percent `100`, plan `inr_90d`, max redemptions = batch size + 10%, and valid-until = the end of the placement season.
3. Build the cohort link: `https://<domain>/app/sign-up?campus=<CAMPUS>&redirect=%2Fapp%2Fbilling%3Fplan%3Dinr_90d%26coupon%3D<CODE>`.
4. Share it through the placement cell together with a one-page how-to. Track it with `/app/admin` (coupons, and the funnel by campus).

**Student side:**

| # | Student | System | Route / endpoint |
|---|---|---|---|
| 1 | Opens the link and signs up | Clerk sign-up; `campus` is carried to onboarding | `/app/sign-up` |
| 2 | Completes onboarding | Campus shown pre-filled; `meInit` sets `campus_code` (only if NULL; it can be set once) | `POST /api/me/init` |
| 3 | Is redirected to billing | `CouponField` pre-filled → `couponPreview` → "₹0 (100% off)" | `/app/billing?plan=inr_90d&coupon=<CODE>` |
| 4 | Clicks "Activate pass" | `couponRedeem` → `grantPass` (provider `coupon`) → `coupon_redeemed` → pass active for 90 days | `POST /api/billing/coupons/redeem` |

**Rules:**
- A 100%-off code can be redeemed whatever the account currency, because nothing is charged (1.10, issue 3).
- Codes below 100% apply only at INR checkout (J6).
- `COUPON_INVALID` reasons map to the messages in 1.5.
- After the pilot, a college pays for a cohort **outside the product**: founder invoice, paid by bank transfer or a Razorpay Payment Link created in the dashboard. The founders then issue a new 100% code for that batch. Such Razorpay payments must be ignored by our webhook (1.10, issue 6).

**Done when:**
- Given a 100% code with 2 redemptions left, then user A activates a pass. User A trying again gets `already_used`. After user B redeems, user C gets `exhausted`.
- `adminCoupons` shows 2 redemptions for the code.

#### J9. Data export and account deletion

| # | User | System | Route / endpoint |
|---|---|---|---|
| 1 | Settings → "Download my data" | `meExport` → `saveBlob` as `data-export-YYYY-MM-DD.json` | `GET /api/me/export` |
| 2 | Settings → "Delete account" | `ConfirmDialog` explains what is erased (all documents, profile, consents, quotas, sign-in account), what is kept pseudonymised (payment records for tax), and that an active pass ends with no refund beyond `/legal/refunds`. The user must type `DELETE`. | — |
| 3 | Confirms | `meDelete {confirm:"DELETE"}` → erasure (Appendix A §2.3) → Clerk user deleted (a failure becomes `clerk_pending` and is retried by `runDaily`) → `account_deleted {hadPass}` | `DELETE /api/me` |
| 4 | — | Full-screen "Your account has been deleted" → `signOutAndClearDrafts()` → "Go to home" → `/` | — |

Signing in again with the same identity later creates a new, empty account with a new free period. This is accepted, because the abuse value is a few cents of AI and Clerk's bot protection applies.

**Done when:**
- Given a user with resumes and a pass, when they delete their account, then signing in again starts onboarding with no documents.
- `deletion-completeness.test.ts` passes.
- `adminOverview` still counts the earlier payment.

#### J10. AI consent withdrawal and pass expiry

- **Withdraw AI consent:** Settings → toggle "AI processing" off → `meConsent {kind:'ai_processing', granted:false}`. Every AI button then shows the consent card, and documents are unaffected.
- **Pass ending:** when `access.tier = 'pass'` and `expiresAt − now ≤ 3 days`, the dashboard shows "Your pass ends on <date>". No email is sent in the MVP. After expiry the free limits apply, nothing is deleted, and count caps block only creation (Appendix A §6.3).

### 1.5 User-facing error handling (all journeys)

`call()` throws `ApiClientError` with the code. One mapping, used everywhere:

| Code | UI | Retry |
|---|---|---|
| `UNAUTHENTICATED` | Silent token refresh once, then redirect to `/app/sign-in?redirect=<current>` | auto once |
| `CONSENT_REQUIRED` | `details.kind` = `terms`/`privacy` → redirect to onboarding. `ai_processing` → inline consent card, then repeat the action. | after consent |
| `VALIDATION_FAILED` | Field errors from `details.issues` next to the inputs | user fixes |
| `NOT_FOUND` | `EmptyState` "This item doesn't exist or was deleted" + link to `/app` | no |
| `REVISION_CONFLICT` | Conflict dialog (J2) | user chooses |
| `PAYWALL` / `QUOTA_EXHAUSTED` | Paywall (free) or reset-date notice (pass/lifetime), J6 table | no |
| `RATE_LIMITED` | "You've used today's free AI tips. The report above still works." + sign-up CTA | next UTC day |
| `TURNSTILE_FAILED` | Reset the widget: "Please complete the check again" | yes |
| `AI_DISABLED` | "AI features are paused right now. Everything else works." | no |
| `AI_BUDGET_EXHAUSTED` | "AI is at capacity for today. Try again after 05:30 IST (00:00 UTC)." | next UTC day |
| `AI_PROVIDER_ERROR` / `AI_OUTPUT_INVALID` | "That didn't work. Your AI quota was not used." + Try again | yes |
| `READ_ONLY` | Banner "Maintenance in progress: your changes are kept on this device and will sync after"; writes are queued locally | auto every 30 s |
| `CHECKOUT_DISABLED` | "Payments in <currency> open soon." Coupon field stays available. | no |
| `PLAN_UNAVAILABLE` | "This plan is no longer available" → reload the plans | no |
| `COUPON_INVALID` | `not_found` "Code not found" · `expired` "This code has expired" · `exhausted` "This code has been fully used" · `already_used` "You've already used this code" · `plan_not_eligible` "This code doesn't apply to this plan" · `not_free` "This code gives a discount; use it at checkout" | user edits |
| `PAYMENT_NOT_VERIFIED` | "We couldn't confirm this payment yet. If money left your account, it will activate automatically within a few minutes, or contact support@<domain>." | no |
| `PAYLOAD_TOO_LARGE` | "This is over the size limit" + the limit | user edits |
| `INTERNAL`, network error | Toast "Something went wrong. Try again." (Sentry, no content) | yes |

### 1.6 MVP scope

#### 1.6.1 In scope

| Capability | Where it is specified | Milestone (Part 10) |
|---|---|---|
| Clerk sign-in (Google, email code, email link), onboarding with consents, currency, campus code | Appendix A §3 `me*`, §4.2 | M1 |
| Master resume editor: all sections in §7.1, drag reorder, hide/rename sections, autosave + local drafts, live PDF preview | Appendix A §7, §4.2 | M1 |
| Deterministic content checks (`RULE_IDS`) | Appendix A §5.2 | M1 |
| Templates: `global_classic` + `india_fresher` (M1); the other 4 (M2) | Appendix A §7.2 | M1/M2 |
| PDF export (M1), DOCX export (M2), soft export gate | Appendix A §3 `exportAuthorize` | M1/M2 |
| Import from PDF/DOCX/paste with AI mapping and section-by-section confirmation | J3 | M2 |
| AI bullet rewrite, summary, cover letter with diff + honesty flags | 1.3, J4, J5b | M2 |
| JD tailoring: tailored versions, cached AI keywords, match %, heuristic fallback | J5 | M2 |
| Quotas, count caps, global and anonymous AI budgets, kill switches | Appendix A §6.3-6.8 | M2 |
| Passes (USD Dodo, INR Razorpay), webhooks, stacking, refunds/disputes → revoke | J6 | M3 |
| Our INR coupons (preview, partial at checkout, 100% redeem) | J8 | M3 |
| Free checker with Turnstile, IP caps, anonymous budget | J1 | M3 |
| Static marketing site: `/`, `/pricing`, `/templates`, `/campus`, `/about`, `/contact`, `/how-match-score-works`, 5 guides, legal pages, licenses, sitemap, robots | Appendix A §4.1 | M1 (legal, pricing, contact) / M4 (rest) |
| Data export, account deletion, AI consent withdrawal | J9, J10 | M4 |
| First-party funnel events, Cloudflare Web Analytics, read-only admin page, ops scripts | Appendix A §6.7, §6.9 | M4 |
| Sentry (scrubbed), backups + restore test, rollback workflow, runbooks | Appendix A §1 | M4 |

#### 1.6.2 Out of scope (MVP)

| Item | Why not now | Revisit when (Part 10.8) |
|---|---|---|
| Carrying the checker's file into `/app/import` | Simplicity and privacy; the user re-uploads | Gate 1 shows checker→signup below 10% |
| Copying tailored improvements back to the master | Merge UX is complex | 20+ beta users ask for it |
| Version history beyond the in-memory undo stack | Storage/D1 writes | Paid users report lost work |
| Public resume links, hosted web resumes, sharing with mentors | Abuse + privacy surface | After Gate 2 |
| Photo, date of birth, gender, marital status, religion, nationality fields | Bias, privacy and ATS concerns (Appendix A §7.1) | Never |
| More than 6 templates, template marketplace, custom template designer | Scope | After Gate 2, data-driven |
| Recurring subscriptions (INR UPI AutoPay/e-mandate, USD auto-renew) | Passes are the positioning; avoids renewal-law exposure and fees | Never in year 1 |
| Transactional email (pass-expiry reminders, receipts from us) | One less service; Clerk and the providers already send email | ≥ 50 paid passes/month (Resend Free) |
| Self-serve college/coach dashboard, team seats, SSO | Coupons + spreadsheet cover the first cohorts | 3+ colleges paying |
| Hindi/regional-language UI | Founders help in Hindi/Hinglish over WhatsApp instead | Clear demand after Gate 2 |
| Job tracker, auto-apply, LinkedIn import or scraping | LinkedIn terms; scope | Never (LinkedIn); tracker after Gate 3 |
| AI mock interviews, any audio/video | Cost and scope | Never in the MVP family |
| Chrome extension, native mobile apps | The web editor is mobile-usable | After Gate 3 |
| Programmatic SEO at scale | Google's scaled-content-abuse policy | Expand only the guides that get impressions |
| Affiliate software | A few creator coupon codes are tracked manually | 10+ active creators |
| Lifetime-deal marketplaces | Margin and support load | Never |
| GST tax invoices | Not GST-registered at launch; a CA advises on the threshold | Before INR turnover nears ₹20 lakh |
| Non-English resumes, right-to-left scripts | Fonts and ATS testing | After Gate 2 |
| Human resume-writing service as a product line | Only the week-1 manual validation offer exists (Part 10.4) | Never as a product |

### 1.7 Success metrics and funnel

#### 1.7.1 Funnel stages

| Stage | Signal | Source |
|---|---|---|
| S0 Visit | Page views per path | Cloudflare Web Analytics (no cookies) |
| S1 Checker use | `checker_used` | `events` (anonymous; counts uses, not people) |
| S2 Checker CTA | `checker_cta_clicked` | `events` |
| S3 Sign-up | `signup_completed` | `events` |
| S4 First resume | first `resume_created` per user | `events` |
| S5 **Activation** | `first_export` | `events` |
| S6 Paywall seen | `paywall_viewed` (by trigger) | `events` |
| S7 Checkout started | `checkout_started` | `events` |
| S8 **Paid** | `payment_succeeded` / non-revoked `passes` with provider `dodo` or `razorpay` | `events`, `passes` |
| S8c Coupon pass (not revenue) | `coupon_redeemed` | `events` |

North-star metric: **paid passes per week** (provider `dodo` + `razorpay`, not revoked, coupons excluded).

#### 1.7.2 Metric definitions (canonical; `adminFunnel`, `adminOverview` and `npm run funnel` must compute exactly these)

`:from` / `:to` are inclusive `DayKey`s. Bounds in ms (`:fromMs`, `:toMs`) are the start of `:from` and the end of `:to`, in UTC.

```sql
-- M1 checker uses
SELECT COUNT(*) FROM events WHERE name = 'checker_used' AND day BETWEEN :from AND :to;
-- M2 checker-attributed sign-ups (proxy; users deleted since then drop out)
SELECT COUNT(*) FROM users WHERE ft_landing_path = '/resume-checker' AND created_at BETWEEN :fromMs AND :toMs;
-- checker_to_signup_pct = 100 * M2 / M1   (a lower bound, because M1 counts uses, not visitors)
-- M3 sign-ups (rows survive deletion because user_id is only nulled)
SELECT COUNT(*) FROM events WHERE name = 'signup_completed' AND day BETWEEN :from AND :to;
-- M4 activated: sign-ups in the window with a first_export within 7 days of sign-up
SELECT COUNT(DISTINCT s.user_id) FROM events s JOIN events e
  ON e.user_id = s.user_id AND e.name = 'first_export' AND e.created_at <= s.created_at + 7*86400000
 WHERE s.name = 'signup_completed' AND s.day BETWEEN :from AND :to;
-- M5 paid users among sign-ups in the window (paid at any time up to now)
SELECT COUNT(DISTINCT p.user_id) FROM passes p JOIN users u ON u.id = p.user_id
 WHERE p.provider IN ('dodo','razorpay') AND p.revoked_at IS NULL AND u.created_at BETWEEN :fromMs AND :toMs;
-- signup_to_paid_pct = 100 * M5 / (users created in the window)
-- M6 paid passes and revenue per currency (net of refunds; tax excluded)
SELECT currency, COUNT(*), SUM(amount_minor) FROM passes
 WHERE provider IN ('dodo','razorpay') AND revoked_at IS NULL AND purchased_at BETWEEN :fromMs AND :toMs GROUP BY currency;
-- M7 LLM cost
SELECT SUM(cost_micro_usd) FROM llm_calls WHERE created_at BETWEEN :fromMs AND :toMs;
-- M8 refund rate = revoked (reason refund|dispute) / all dodo+razorpay passes purchased in the window
```

**Revenue across currencies:** gate reviews convert INR at a fixed planning rate of **₹88 = $1**. The founders re-check this rate each quarter in the gate review notes. Code never converts currencies; admin shows each currency separately.

#### 1.7.3 Targets

| Metric | Day 30 (Tue 2026-11-03) | Day 90 (Sat 2027-01-02) | Day 180 (Fri 2027-04-02) | Gate use |
|---|---|---|---|---|
| Paid passes (cumulative, non-revoked) | ≥ 10 | ≥ 60 | ≥ 200 | Gate 1 |
| Checker → sign-up % (M2/M1, trailing 30 d) | ≥ 10% | ≥ 12% | ≥ 12% | Gate 1 |
| Sign-up → paid % (cumulative) | — | ≥ 1% (assessed after 1,000 sign-ups) | ≥ 2% | Gate 2 |
| Trailing-30-day revenue (USD-equivalent) | — | ≥ $300 | ≥ $800 and growing month over month | Gates 2, 3 |
| Colleges paying for a cohort | — | ≥ 1 | ≥ 3 | Gate 2 |
| Activation (M4/M3) | ≥ 40% | ≥ 45% | ≥ 50% | Health |
| Median time from sign-up to first export | ≤ 20 min | ≤ 15 min | ≤ 15 min | Health |

The gates and the actions they trigger are in Part 10.6.

#### 1.7.4 Guardrail metrics (checked weekly; a red value pauses growth work until fixed)

| Guardrail | Green | Red | Source |
|---|---|---|---|
| LLM cost as % of revenue (trailing 30 d) | ≤ 10% | > 25% | M7 / M6 |
| Payment fees as % of revenue | ≤ 8% | > 12% | provider dashboards |
| Refund + dispute rate | ≤ 5% | > 10% | M8 |
| Days `ai_daily_budget_micro_usd` was reached (trailing 7 d) | 0 | ≥ 3 | `adminCosts` |
| Complaints about invented content in AI output | 0 | ≥ 1 (investigate the same day) | support inbox |
| Unresolved Sentry `error` issues older than 48 h | 0 | ≥ 3 | Sentry |
| Support reply time | < 24 h | > 72 h | support inbox |

### 1.8 Glossary

| Term | Meaning |
|---|---|
| Master resume | A row in `resumes`: the user's main resume. Tailored versions are copied from it. |
| Tailored version | A row in `tailored_versions`: a full copy of a master's data plus one JD. Editing it never changes the master. |
| JD | Job description text pasted by the user (1-15,000 characters). |
| Keyword match % | The published formula in Appendix A §5.2 over required (weight 2) and preferred (weight 1) keywords. It is never called an "ATS score". |
| Honesty source | The text an AI output may draw facts from: the stored document plus user-authored input. Never JD text (1.3). |
| Honesty flag | A `HonestyFlag`: a span in AI output (number, percent, money, organization, degree, skill, date) not found in the honesty source. |
| Keep anyway | The user's confirmation that a flagged value is true. It is held only in memory. |
| AI action | One `ai_action` quota unit: one bullet rewrite, summary or cover letter request (up to 3 candidates). |
| Pass | A one-time purchase giving a tier for a fixed number of days (`inr_7d`, `inr_30d`, `inr_90d`, `usd_30d`, `usd_90d`). It does not auto-renew, and passes stack end to end. |
| Lifetime (founder) | `inr_lifetime` / `usd_lifetime`: no end date, monthly fair-use limits, capped by `lifetime_founder_limit`. |
| Tier | `free` \| `pass` \| `lifetime`, from `computeAccess` (Appendix A §6.4). |
| Limit group | The row group in `plan_limits` used for the caller's limits (`free`, `pass_7d`, `pass_30d`, `pass_90d`, `lifetime`). |
| Free period | A 30-day window anchored on `users.created_at`. The free quotas reset at each boundary. |
| Quota feature / count cap | A per-period counter (`QuotaFeature`) / a maximum number of stored rows (`CountCap`). |
| Paywall | `PaywallDialog`, shown to free users when a limit is reached (J6). |
| Soft export gate | `exportAuthorize`: counts exports before browser rendering. It cannot technically stop a determined user. |
| Intent | A `checkout_intents` row created for each checkout attempt. Its status is `created` → `paid`/`failed`/`expired`. |
| Grant / revoke | Inserting a pass (`grantPass`) or marking it revoked (`revokePass`). These are the only ledger writes. |
| Coupon | Our INR-only code (`coupons`). 100% codes are redeemed without checkout; partial codes reduce the Razorpay amount. Dodo has its own discount codes for USD. |
| Campus code | An uppercase code stored once on `users.campus_code`, used to report the funnel by college. |
| Cohort | One college batch that gets passes through one coupon code. |
| Checker | The anonymous `/resume-checker` page (J1). |
| Kill switch | `ai_enabled`, `anon_ai_enabled`, `checkout_*_enabled` and `read_only` in `app_settings`. Founders change them with npm scripts, without a deploy. |
| Kill gate | A dated business checkpoint with a pass/fail rule (Part 10.6). |
| MoR | Merchant of record: Dodo sells USD passes in its own name and handles sales tax/VAT. |
| `[BRAND]` | The placeholder value of `BRAND_NAME`. Production deploys are blocked while `BRAND_IS_PLACEHOLDER` is true. |

### 1.9 SKELETON ADDITIONS

| Name | Kind | Lives in | Why |
|---|---|---|---|
| `safeRedirectPath(raw: string \| null): string` | pure function | `src/shared/config/routes.ts` | Prevents open redirects on the `redirect` (sign-in/up) and `next` (onboarding) query params. It returns `raw` only when its path part equals an `APP_ROUTES` path, it starts with `/app`, and it has no scheme, `//` or backslash. Otherwise it returns `'/app'`. Unit-tested in `tests/unit/shared/routes.test.ts`. |
| `defaultsForUser(currency: Currency, country: string \| null): { templateId: TemplateId; locale: 'en-IN' \| 'en-US' \| 'en-GB'; pageSize: 'A4' \| 'LETTER'; sample: 'global' \| 'india' }` | pure function | `src/shared/config/templates.ts` | One source for the defaults table in 1.2.4, used by `DashboardPage` (blank/sample) and `ImportPage` (settings of an imported draft). |
| `tests/unit/shared/routes.test.ts` | unit test file | `tests/unit/shared/` | Covers `safeRedirectPath` (open-redirect cases). |

### 1.10 SKELETON ISSUES

1. **First-touch capture on static pages.** `captureFirstTouch()` is listed only in `src/app/lib/attribution.ts`. Guides, `/`, `/pricing` and `/resume-checker` are static Astro pages, so visitors who land there would never record `ft_landing_path` or UTMs, and checker→signup (1.7.2, M2) would read zero. Fix: `BaseLayout.astro` includes a bundled `<script>` that imports `@app/lib/attribution` and calls `captureFirstTouch()` on every static page. The name is unchanged.
2. **Honesty source vs "the request's own input text" (§7.3).** For `aiBullet` with JD keywords and for `aiCoverLetter`, the JD text and keywords must **not** count as source. Otherwise a JD skill copied into a bullet is never flagged, which breaks the core promise. This part assumes that "input text" means user-authored text only: the original bullet or summary, and for a cover letter the tailored row's `job_title` and `company`. JD body text and keywords are excluded.
3. **Coupon redemption vs account currency.** The skeleton does not say whether `couponRedeem` requires `users.currency = 'INR'`. This part assumes it does not, because a 100%-off grant charges nothing and beta testers in other countries are onboarded with coupons (Part 10, M5). The billing section must confirm this.
4. **`PAYWALL.details.reason` values are not enumerated.** This part derives the `paywall_viewed` trigger from the UI action (J6 table), so the client does not depend on them. The API section should still enumerate them (suggested: `free_tier`, `count_cap`, `zero_limit`).
5. **Export order.** `ExportDialog` is specified as `POST /api/exports` first, then render. If rendering then fails in the browser, the user loses export quota (the free tier has only 1 PDF per period) with no refund path. Suggestion: render first, call `exportAuthorize`, then `saveBlob` only when authorized. J7 follows the skeleton order until the documents section decides.
6. **Payments that are not ours on the Razorpay account.** Cohort fees collected with dashboard Payment Links (J8) fire `payment.captured` on the same webhook. The webhook flow must record such events as `ignored` (no `intent_id` in notes, no matching `checkout_intents` row) and never grant or reject noisily.
7. **The checker's deterministic report and `runContentChecks`.** `runContentChecks` takes a `ResumeData`, but the checker only has plain text. The checker section must define how the anonymous report is computed (text-level subset of the rules, or a heuristic parse), and it must stay deterministic and run in the browser. J1 depends only on "renders without a network call".
