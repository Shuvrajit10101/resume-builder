# Part 00: Canonical skeleton

*Status: binding for all spec sections (parts 01-10). Written 2026-10-03. It derives from ADR 0001 (`docs/decisions/0001-architecture.md`, binding), and the product memo (`research/Simple_SaaS_Pick_AI_Resume_Builder.md`). Where the two differ, the ADR wins. Examples: free AI quota is 5 AI actions + 1 AI import + 1 JD analysis per 30 days, not 3 rewrites per day; sign-in is Clerk, not Supabase.*

**Rules for section writers**

1. Use the names in this file exactly: folders, files, tables, columns, endpoint keys, paths, ids, event names, settings keys, env var names and error codes. Do not rename anything or add synonyms.
2. If your section needs a name that is not listed here, add it in an "Additions to skeleton" table at the end of your part. Give the name, its kind, where it lives and why it is needed. An addition must not overlap or conflict with anything here.
3. Never write a product name. Use `BRAND_NAME` (from `src/shared/config/brand.ts`) in code and `{BRAND_NAME}` in copy. Never write "Rezi" anywhere: not in code, copy, templates, content, tests or comments (`npm run check:brand` fails the build).

---

## 0. Global conventions

| Topic | Rule |
|---|---|
| Language and tooling | TypeScript only (strict, `noUncheckedIndexedAccess`). npm with exact pinned versions. Node 22 (from `.nvmrc`) for tooling. One package, no monorepo. |
| Path aliases | `@shared/*` → `src/shared/*`, `@server/*` → `src/server/*`, `@app/*` → `src/app/*`, `@documents/*` → `src/documents/*`, `@ui/*` → `src/ui/*` |
| File names | `.ts` files are kebab-case. React component files are PascalCase `.tsx`. Unit and integration tests are `*.test.ts`; Playwright tests are `*.spec.ts`. |
| Row ids | UUIDv7 strings from `newId()` in `src/server/core/ids.ts`. All ids are generated on the server. Path params are validated with `UuidV7Schema`. |
| Item ids inside resume JSON | `crypto.randomUUID()`, generated in the browser. Up to 40 characters. |
| Time | Unix epoch **milliseconds**, UTC, stored as `INTEGER` in columns named `*_at`. Day keys are `'YYYY-MM-DD'` UTC strings and month keys are `'YYYY-MM'` UTC strings. No `Date` objects in the data layer. Server code reads the time only through `now()` from `src/server/core/clock.ts`. |
| Money | Integer minor units (`amount_minor`: paise or cents) plus a `currency` of `'INR' \| 'USD'`. LLM cost is integer **micro-USD** (`*_micro_usd`). Never use floats for money. |
| SQL | Tables are snake_case plural and columns are snake_case. Types are `TEXT` and `INTEGER` only. Booleans are `INTEGER` 0/1 with a CHECK constraint. JSON is stored as `TEXT`. Enum columns get a CHECK constraint. No SQL `FOREIGN KEY` constraints: integrity is enforced by the repositories and the deletion/export completeness tests. |
| Drizzle | Every column is declared with an explicit snake_case name (`text('user_id')`) and a camelCase TS key (`userId`). |
| API envelope | Success: `{ "data": T }`. Error: `{ "error": { "code": ErrorCode, "message": string, "details"?: object } }`. JSON only. |
| Auth transport | `Authorization: Bearer <Clerk session JWT>` (from Clerk `getToken()`). Server routes never read cookies. |
| CSRF | Every non-GET route except webhooks requires `Origin === SITE_ORIGIN` and `Content-Type: application/json`. |
| Body limits | Default 128 KB. `/api/ai/import` and `/api/checker/tips`: 100 KB. Webhooks: 64 KB, read raw. |
| AI output schemas | OpenAI strict structured outputs: every property is required and optional values are `nullable`. Never use `.optional()` in `src/shared/schemas/ai.ts`. |
| Logging | Only through `log` (`src/server/core/logger.ts`) on the server and `src/app/lib/logger.ts` in the browser. Never log request bodies, resume fields, JD text, prompts, emails or raw IPs. |

### Error codes (`src/shared/schemas/errors.ts`)

| Code | HTTP | When |
|---|---|---|
| `UNAUTHENTICATED` | 401 | Missing, expired or invalid Clerk JWT |
| `FORBIDDEN` | 403 | Admin route called by a non-admin |
| `BAD_ORIGIN` | 403 | Origin check failed |
| `UNSUPPORTED_MEDIA_TYPE` | 415 | Non-JSON body on a JSON route |
| `PAYLOAD_TOO_LARGE` | 413 | Body over the route's limit, or resume JSON over 65,536 bytes |
| `VALIDATION_FAILED` | 400 | zod validation failed on params, query or body (`details.issues`) |
| `NOT_FOUND` | 404 | Unknown resource, **including another user's id** |
| `REVISION_CONFLICT` | 409 | `baseRevision` is not the stored revision (`details.currentRevision`) |
| `PAYWALL` | 402 | Paid route called on the free tier, or a count cap reached (`details.reason`, `details.cap`) |
| `QUOTA_EXHAUSTED` | 429 | Period quota used up (`details.feature`, `limit`, `periodEndsAt`) |
| `RATE_LIMITED` | 429 | Per-IP daily cap reached |
| `CONSENT_REQUIRED` | 403 | Terms/privacy consent missing for a write, or `ai_processing` consent missing for AI (`details.kind`) |
| `TURNSTILE_FAILED` | 403 | Turnstile siteverify failed |
| `AI_DISABLED` | 503 | `ai_enabled` or `anon_ai_enabled` is false |
| `AI_BUDGET_EXHAUSTED` | 503 | Global or anonymous daily USD budget reached |
| `AI_PROVIDER_ERROR` | 502 | OpenAI error or timeout; the reservation is refunded |
| `AI_OUTPUT_INVALID` | 502 | Model output failed zod validation; the reservation is refunded |
| `READ_ONLY` | 503 | `read_only` setting is true (all non-GET user/paid routes) |
| `CHECKOUT_DISABLED` | 503 | `checkout_usd_enabled` / `checkout_inr_enabled` is false |
| `PLAN_UNAVAILABLE` | 409 | Lifetime founder cap reached, or the plan does not match the currency |
| `COUPON_INVALID` | 400 | `details.reason`: `not_found \| expired \| exhausted \| already_used \| plan_not_eligible \| not_free` |
| `PAYMENT_NOT_VERIFIED` | 400 | Razorpay callback signature or re-fetch check failed |
| `WEBHOOK_SIGNATURE_INVALID` | 401 | Webhook signature or timestamp check failed (the provider retries) |
| `INTERNAL` | 500 | Unhandled error; reported to Sentry with no body |

---

## 1. Repository tree

The repo is named `resume-builder`: a private GitHub repo with one package.

```
resume-builder/
├── CLAUDE.md                         # invariants from Part 8 + how to run checks; Claude Code reads it every session
├── README.md                         # founder-facing: what each npm script does, links to docs/runbooks
├── package.json                      # "type": "module", exact versions, scripts listed in Part 6.9
├── package-lock.json
├── .nvmrc                            # 22
├── .gitignore                        # dist/, .wrangler/, .dev.vars, .env, node_modules/, playwright-report/, test-results/, *.age, *.sql.gz
├── .env.example                      # PUBLIC_* build-time vars for local Astro builds
├── .dev.vars.example                 # every Worker secret name with a local/mock value
├── .gitleaks.toml
├── .prettierrc.json
├── tsconfig.json                     # strict + path aliases (Part 0)
├── astro.config.ts                   # output 'static', build.format 'file', trailingSlash 'never', integrations: react(), sitemap()
├── wrangler.jsonc                    # Worker, static assets, D1 bindings, cron; top level = local, env.staging, env.production
├── drizzle.core.config.ts            # schema src/server/data/schema/core.ts → out migrations/core, dialect sqlite, driver d1-http
├── drizzle.docs.config.ts            # schema src/server/data/schema/docs.ts → out migrations/docs
├── eslint.config.js                  # typescript-eslint, import boundaries (Part 8), no-console, no-restricted-imports
├── vitest.config.ts                  # unit project: tests/unit/** (node environment)
├── vitest.workers.config.ts          # integration project: tests/integration/** + tests/generated/** via @cloudflare/vitest-pool-workers
├── playwright.config.ts              # projects: golden (local preview), e2e-staging, smoke (BASE_URL)
├── components.json                   # shadcn/ui config → src/ui
├── .claude/
│   └── settings.json                 # deny rules: prod deploy, remote D1 writes, `wrangler secret put`, force-push
├── .github/
│   ├── pull_request_template.md      # checklist: tests added, sensitive paths touched? second review done?
│   └── workflows/
│       ├── ci.yml                    # every push/PR: typecheck, lint, unit, integration, generated matrices, check:*, gitleaks, build, bundle scan, golden
│       ├── deploy-staging.yml        # on green CI for main: build (staging vars) → d1 migrations apply (staging) → deploy → e2e-staging
│       ├── deploy-production.yml     # workflow_dispatch(sha): check:brand --prod → Time Travel bookmarks → migrations → deploy → smoke → auto `wrangler rollback` on failure
│       ├── rollback.yml              # workflow_dispatch(env, version | 'previous', optional d1 restore: db + bookmark + typed confirmation)
│       └── backup.yml                # cron 22:00 UTC (03:30 IST): d1 export core-prod + docs-prod → age encrypt → artifact (14 d) → restore test
├── docs/
│   ├── decisions/0001-architecture.md
│   ├── superpowers/specs/            # this spec (parts/00-skeleton.md … parts/10-*.md)
│   └── runbooks/
│       ├── accounts-setup.md         # Cloudflare, Clerk, OpenAI, Dodo, Razorpay, Turnstile, Sentry, GitHub, Email Routing
│       ├── secrets.md                # how to set and rotate every secret in Part 6.6
│       ├── deploy-and-rollback.md
│       ├── restore-core.md           # Time Travel restore of core + npm run replay-payments
│       ├── restore-docs.md
│       ├── ai-kill-switch.md
│       └── incident-data-leak.md     # DPDP/GDPR breach steps
├── migrations/
│   ├── core/
│   │   ├── 0000_init.sql             # all core tables (Part 2.2)
│   │   ├── 0001_seed_defaults.sql    # custom migration: plan_limits + app_settings defaults (Part 6)
│   │   └── meta/                     # drizzle-kit journal (wrangler ignores it)
│   └── docs/
│       ├── 0000_init.sql             # all docs tables (Part 2.3)
│       └── meta/
├── public/
│   ├── _headers                      # security headers for static assets (CSP, HSTS, nosniff, Referrer-Policy, Permissions-Policy)
│   ├── favicon.svg
│   ├── fonts/                        # self-hosted OFL fonts: NotoSans-{Regular,Italic,Bold,BoldItalic}.ttf, NotoSerif-{Regular,Italic,Bold,BoldItalic}.ttf (+ .woff2 for web)
│   └── og/                           # Open Graph images
├── scripts/
│   ├── lib/d1.ts                     # runSql(env, db, sql, params): wraps `wrangler d1 execute <db> --env <env> --remote|--local --json`
│   ├── lib/args.ts                   # parses --env local|staging|production (required, no default)
│   ├── ai-switch.ts                  # npm run ai-off | ai-on
│   ├── set-limit.ts                  # npm run set-limit
│   ├── set-setting.ts                # npm run set-setting
│   ├── funnel.ts                     # npm run funnel
│   ├── costs.ts                      # npm run costs (LLM spend vs budget, D1 write estimate)
│   ├── coupon-create.ts              # npm run coupon:create
│   ├── coupon-list.ts                # npm run coupon:list
│   ├── replay-payments.ts            # re-delivers provider payments since a timestamp as signed webhooks (secrets from shell env only)
│   ├── check-migrations.ts           # additive-only: no DROP/RENAME/table rebuild; applied files unchanged vs origin/main
│   ├── check-boundaries.ts           # grep rules from Part 8 (DB bindings, ledger, OpenAI URL, console)
│   ├── check-bundle-secrets.ts       # scans dist/ for secret patterns (sk-, rzp_live_ secret shapes, whsec_, PEM, sk_live_/sk_test_)
│   ├── check-brand.ts                # fails on /\brezi\b/i in src/, public/, content, tests; --prod also fails if BRAND_IS_PLACEHOLDER
│   ├── gen-licenses.ts               # writes src/generated/licenses.json (third-party notices) before build
│   ├── seed-local.ts                 # local D1: 2 users, sample resumes, one pass, one coupon
│   └── backup-restore-test.sh        # decrypt-free restore test: imports the plaintext dump in CI before encryption, checks row counts
├── src/
│   ├── worker.ts                     # Worker entry: fetch → Hono app for /api/*; scheduled → jobs/daily
│   ├── env.d.ts                      # Astro ImportMetaEnv types for PUBLIC_* vars
│   ├── content.config.ts             # collections: guides, legal
│   ├── generated/licenses.json       # build output of gen-licenses (git-ignored)
│   ├── content/
│   │   ├── guides/*.md               # SEO guides (Part 4.1)
│   │   └── legal/{terms,privacy,refunds}.md
│   ├── pages/                        # Astro routes (Part 4.1)
│   ├── layouts/                      # BaseLayout.astro, MarketingLayout.astro, LegalLayout.astro, GuideLayout.astro, AppShellLayout.astro
│   ├── components/                   # Astro components: Header.astro, Footer.astro, Seo.astro, TemplateCard.astro, FaqList.astro
│   ├── islands/                      # React islands for static pages: CheckerApp.tsx, PricingTable.tsx, CurrencySwitch.tsx
│   ├── ui/                           # shadcn/ui: components/ui/*.tsx, lib/cn.ts (shared by islands and app)
│   ├── styles/global.css             # Tailwind v4 (@import "tailwindcss"), design tokens
│   ├── app/                          # React 19 SPA mounted at /app
│   │   ├── App.tsx                   # ClerkProvider, QueryClientProvider (TanStack Query v5), RouterProvider, ErrorBoundary
│   │   ├── router.tsx                # React Router (library mode) route table from APP_ROUTES
│   │   ├── guards.tsx                # RequireAuth, RequireOnboarded, RequireAdmin
│   │   ├── lib/
│   │   │   ├── api.ts                # call(contract, {params,query,body}) → typed data; throws ApiClientError
│   │   │   ├── auth.ts               # useSessionToken(), signOutAndClearDrafts()
│   │   │   ├── query-keys.ts
│   │   │   ├── drafts.ts             # IndexedDB (idb): getDraft, putDraft, deleteDraft, clearAllDrafts
│   │   │   ├── autosave.ts           # useAutosave(): 15 s idle, blur, visibilitychange=hidden; skip if content hash unchanged
│   │   │   ├── analytics.ts          # trackPublic(name, props) → POST /api/events; trackUser(name, props) → POST /api/me/events
│   │   │   ├── attribution.ts        # captureFirstTouch(), readFirstTouch() (localStorage key 'ft_v1')
│   │   │   ├── sentry.ts             # @sentry/react init: sendDefaultPii false, no replay, beforeSend scrubber
│   │   │   └── logger.ts             # the only console.* user in browser code
│   │   ├── state/
│   │   │   ├── editor-store.ts       # Zustand: doc, docKind, revision, dirty, selection, in-memory undo stack
│   │   │   └── ai-store.ts           # pending AI suggestions, honesty flags, per-flag "keep anyway" state
│   │   ├── routes/                   # one component per SPA route (Part 4.2)
│   │   ├── editor/
│   │   │   ├── sections/             # BasicsForm, SummaryForm, ExperienceList (experience/internships/positions), EducationList,
│   │   │   │                         # ProjectsList, SkillsForm, CertificationsList, AchievementsList, LanguagesList, CustomSectionList
│   │   │   ├── SectionOrder.tsx      # @dnd-kit/core + @dnd-kit/sortable
│   │   │   ├── PreviewPane.tsx       # live PDF preview (documents/preview.ts)
│   │   │   ├── ChecksPanel.tsx       # rules.runContentChecks
│   │   │   ├── MatchPanel.tsx        # keywords.computeMatch (tailored docs only)
│   │   │   ├── AiSuggestPanel.tsx
│   │   │   ├── DiffView.tsx          # jsdiff side-by-side
│   │   │   ├── HonestyFlagList.tsx
│   │   │   ├── TemplatePicker.tsx
│   │   │   └── ExportDialog.tsx      # POST /api/exports then documents/pdf|docx
│   │   ├── billing/                  # PlanPicker.tsx, CouponField.tsx, RazorpayCheckout.tsx (loads checkout.js), PaywallDialog.tsx
│   │   └── components/               # AppHeader.tsx, Banner.tsx, QuotaMeter.tsx, ConfirmDialog.tsx, EmptyState.tsx
│   ├── documents/                    # browser-only document engine (used by src/app and src/islands)
│   │   ├── fonts.ts                  # registerPdfFonts(), FONT_FAMILIES
│   │   ├── templates/
│   │   │   ├── index.ts              # TEMPLATE_COMPONENTS: Record<TemplateId, ResumeTemplate>
│   │   │   ├── primitives.tsx        # shared react-pdf blocks: Page, SectionHeading, ItemHeader, BulletList
│   │   │   ├── global-classic.tsx
│   │   │   ├── global-modern.tsx
│   │   │   ├── global-compact.tsx
│   │   │   ├── india-fresher.tsx
│   │   │   ├── india-tech-fresher.tsx
│   │   │   ├── india-campus-academic.tsx
│   │   │   └── cover-letter.tsx      # cover letter PDF layout, styled to match the chosen template family
│   │   ├── pdf.ts                    # renderResumePdf(data), renderCoverLetterPdf(letter, basics, templateId), countPages(blob)
│   │   ├── docx.ts                   # renderResumeDocx(data), renderCoverLetterDocx(letter, basics)
│   │   ├── preview.ts                # drawPdf(blob, container): pdfjs-dist canvas rendering
│   │   ├── import/
│   │   │   ├── index.ts              # extractText(file): dispatch by type, 20,000-char cap, scanned-PDF detection
│   │   │   ├── pdf-text.ts           # extractPdfText(file)
│   │   │   └── docx-text.ts          # extractDocxText(file) (mammoth)
│   │   └── download.ts               # saveBlob(blob, filename), exportFileName(basics, kind, format)
│   ├── server/                       # Worker-only code (Part 5.1)
│   │   ├── app.ts
│   │   ├── env.ts
│   │   ├── core/        route.ts  auth.ts  http.ts  context.ts  logger.ts  sentry.ts  crypto.ts  ids.ts  clock.ts
│   │   ├── data/        db.ts  schema/core.ts  schema/docs.ts  scoped.ts  system.ts  ledger.ts  admin.ts  account-data.ts
│   │   ├── services/    index.ts  account.ts  access.ts  quota.ts  settings.ts  documents.ts  exports.ts  events.ts
│   │   │                abuse.ts  checker.ts  meta.ts  admin.ts
│   │   ├── billing/     provider.ts  products.ts  dodo.ts  razorpay.ts  checkout.ts  coupons.ts  grant.ts  webhooks.ts
│   │   ├── ai/          llm-config.ts  provider.ts  openai.ts  mock.ts  cost.ts  guard.ts
│   │   │   └── tasks/   import-map.ts  jd-keywords.ts  bullet-rewrite.ts  summary.ts  cover-letter.ts  checker-tips.ts
│   │   ├── routes/      index.ts  public.ts  webhooks.ts  me.ts  resumes.ts  tailored.ts  cover-letters.ts  ai.ts
│   │   │                exports.ts  billing.ts  admin.ts
│   │   └── jobs/        daily.ts
│   └── shared/                       # pure, runtime-agnostic (no DOM, no Workers APIs) (Part 5.2)
│       ├── config/      brand.ts  plans.ts  limits.ts  settings.ts  ai-features.ts  templates.ts  routes.ts  legal.ts  input-limits.ts
│       ├── api/         contract.ts
│       ├── schemas/     common.ts  resume.ts  api.ts  ai.ts  events.ts  errors.ts
│       ├── access/      compute-access.ts
│       ├── honesty/     index.ts  source.ts  extract.ts  compare.ts
│       ├── keywords/    index.ts  normalize.ts  match.ts  heuristic.ts  synonyms.ts
│       ├── rules/       index.ts  types.ts  checks/<rule-id>.ts (one file per RuleId)
│       ├── dictionaries/ skills.ts  degrees.ts  verbs.ts  organizations.ts  pronouns.ts
│       ├── text/        normalize.ts  resume-text.ts
│       ├── samples/     sample-resume.ts (fictional person, used by "start from sample" and tests)
│       ├── money.ts
│       └── dates.ts
└── tests/
    ├── unit/
    │   ├── shared/     honesty.test.ts  keywords.test.ts  rules.test.ts  compute-access.test.ts (fast-check)
    │   │               resume-schema.test.ts  money.test.ts  dates.test.ts  plans.test.ts
    │   ├── server/     cost.test.ts  quota-period.test.ts  dodo-signature.test.ts  razorpay-signature.test.ts
    │   │               ids.test.ts  crypto.test.ts  llm-config.test.ts  logger-redaction.test.ts
    │   └── documents/  docx.test.ts  import-text.test.ts
    ├── integration/    me.test.ts  resumes.test.ts  tailored.test.ts  cover-letters.test.ts  ai.test.ts  exports.test.ts
    │                   checker.test.ts  billing-checkout.test.ts  webhooks-dodo.test.ts  webhooks-razorpay.test.ts
    │                   coupons.test.ts  admin.test.ts  account-deletion.test.ts  read-only.test.ts  jobs-daily.test.ts
    ├── generated/      route-manifest.test.ts  auth-matrix.test.ts  idor-matrix.test.ts  contract-coverage.test.ts
    │                   deletion-completeness.test.ts  export-completeness.test.ts
    ├── e2e/            smoke.spec.ts  editor-flow.spec.ts  checker.spec.ts  billing-test-mode.spec.ts  templates-golden.spec.ts
    ├── fixtures/       resumes/*.json  jds/*.txt  files/*.pdf|*.docx  webhooks/dodo/*.json  webhooks/razorpay/*.json  llm/<feature>/*.json
    ├── helpers/        auth.ts (signTestJwt with a test RSA key)  factories.ts  db.ts (apply migrations, reset)  request.ts
    └── snapshots/      route-manifest.json
```

---

## 2. Databases, bindings and tables

### 2.1 Environments, Workers and D1

| Environment | Worker name | Host | Core DB | Docs DB | Bindings |
|---|---|---|---|---|---|
| local | `rb-local` (wrangler dev) | `http://localhost:8787` (API), `http://localhost:4321` (Astro dev) | `rb-core-local` | `rb-docs-local` | `DB_CORE`, `DB_DOCS`, `ASSETS` |
| staging | `rb-staging` | `https://staging.<domain>` (noindex) | `rb-core-staging` | `rb-docs-staging` | same |
| production | `rb-prod` | `https://<domain>` (apex) | `rb-core-prod` | `rb-docs-prod` | same |

- Remote databases are created with `wrangler d1 create <name> --location apac`. They use 4 of the 10 free D1 databases. Database ids are committed in `wrangler.jsonc` (they are not secrets).
- `migrations_dir`: `migrations/core` for `DB_CORE` and `migrations/docs` for `DB_DOCS`. Migrations are applied with `wrangler d1 migrations apply <db> --env <env> --remote`.
- Static assets: `assets: { directory: "./dist", binding: "ASSETS", run_worker_first: ["/api/*"], not_found_handling: "404-page" }`.
- Cron: `triggers.crons: ["30 21 * * *"]` (03:00 IST) for staging and production. It runs `jobs/daily.ts`.
- `compatibility_flags: ["nodejs_compat"]` and `observability.enabled: true`.
- The vitest workers pool uses in-memory D1 with the same binding names and applies both migration folders.

### 2.2 Core database (`DB_CORE`)

```sql
CREATE TABLE users (
  id               TEXT PRIMARY KEY,                 -- UUIDv7 (internal user id used everywhere)
  clerk_user_id    TEXT NOT NULL UNIQUE,             -- JWT sub
  email            TEXT,                             -- lowercased, from session claim 'email'; NULL until known
  country          TEXT,                             -- ISO 3166-1 alpha-2 at creation (request.cf.country)
  currency         TEXT NOT NULL CHECK (currency IN ('USD','INR')),  -- default: 'INR' if country = 'IN' else 'USD'
  campus_code      TEXT,                             -- uppercase; settable once
  ft_utm_source    TEXT,                             -- first-touch attribution, set once by /api/me/init
  ft_utm_medium    TEXT,
  ft_utm_campaign  TEXT,
  ft_landing_path  TEXT,
  ft_referrer_host TEXT,
  created_at       INTEGER NOT NULL,                 -- anchors the free-tier quota period
  last_seen_day    TEXT NOT NULL                     -- 'YYYY-MM-DD'; written at most once per day
);

CREATE TABLE consents (                             -- append-only; the latest row per (user_id, kind) wins
  id         TEXT PRIMARY KEY,
  user_id    TEXT NOT NULL,
  kind       TEXT NOT NULL CHECK (kind IN ('terms','privacy','ai_processing')),
  version    TEXT NOT NULL,                          -- LEGAL_VERSIONS[kind] at the time ('YYYY-MM-DD')
  granted    INTEGER NOT NULL CHECK (granted IN (0,1)),
  created_at INTEGER NOT NULL
);
CREATE INDEX consents_user_kind ON consents (user_id, kind, created_at);

CREATE TABLE passes (                               -- append-only ledger; INSERT only from billing/grant.ts
  id                  TEXT PRIMARY KEY,
  user_id             TEXT NOT NULL,                 -- replaced with 'del_<uuidv7>' on account deletion
  plan_id             TEXT NOT NULL,                 -- PlanId (Part 6.1)
  provider            TEXT NOT NULL CHECK (provider IN ('dodo','razorpay','coupon')),
  provider_payment_id TEXT NOT NULL UNIQUE,          -- Dodo payment_id | Razorpay pay_… | 'coupon:<coupon_redemptions.id>'
  intent_id           TEXT,                          -- checkout_intents.id (NULL for coupon)
  coupon_id           TEXT,
  amount_minor        INTEGER NOT NULL,              -- amount the customer paid for the product, excluding tax
  tax_minor           INTEGER NOT NULL DEFAULT 0,    -- tax collected by the MoR (Dodo); 0 for Razorpay/coupon
  currency            TEXT NOT NULL CHECK (currency IN ('USD','INR')),
  purchased_at        INTEGER NOT NULL,              -- provider payment time
  revoked_at          INTEGER,                       -- the only column ever updated (once, NULL → value)
  revoke_reason       TEXT CHECK (revoke_reason IN ('refund','dispute','fraud')),
  pseudonymised_at    INTEGER
);
CREATE INDEX passes_user ON passes (user_id);

CREATE TABLE checkout_intents (
  id                TEXT PRIMARY KEY,                -- intent_id sent to providers as metadata/notes
  user_id           TEXT NOT NULL,
  plan_id           TEXT NOT NULL,
  provider          TEXT NOT NULL CHECK (provider IN ('dodo','razorpay')),
  currency          TEXT NOT NULL CHECK (currency IN ('USD','INR')),
  list_amount_minor INTEGER NOT NULL,                -- catalog price
  amount_minor      INTEGER NOT NULL,                -- after our coupon (Razorpay) or list price (Dodo)
  coupon_id         TEXT,
  provider_ref      TEXT UNIQUE,                     -- Dodo checkout session id | Razorpay order_…
  status            TEXT NOT NULL CHECK (status IN ('created','paid','failed','expired')),
  pass_id           TEXT,
  created_at        INTEGER NOT NULL,
  updated_at        INTEGER NOT NULL
);
CREATE INDEX checkout_intents_user ON checkout_intents (user_id, created_at);

CREATE TABLE webhook_events (                       -- dedupe + audit; no payload stored
  provider    TEXT NOT NULL CHECK (provider IN ('dodo','razorpay')),
  event_id    TEXT NOT NULL,                         -- Dodo 'webhook-id' header | Razorpay 'X-Razorpay-Event-Id' header
  event_type  TEXT NOT NULL,
  payment_id  TEXT,
  outcome     TEXT NOT NULL CHECK (outcome IN ('processing','granted','revoked','duplicate','ignored','rejected')),
  received_at INTEGER NOT NULL,
  PRIMARY KEY (provider, event_id)
);

CREATE TABLE coupons (                              -- our own codes, INR/Razorpay only (Dodo uses its own discount codes)
  id              TEXT PRIMARY KEY,
  code            TEXT NOT NULL UNIQUE,              -- uppercase A-Z0-9, 4-20 chars
  college         TEXT,
  percent_off     INTEGER NOT NULL CHECK (percent_off BETWEEN 1 AND 100),
  plan_ids        TEXT NOT NULL,                     -- JSON array of eligible INR PlanIds
  max_redemptions INTEGER NOT NULL,
  redemptions     INTEGER NOT NULL DEFAULT 0,
  valid_until     INTEGER NOT NULL,
  active          INTEGER NOT NULL DEFAULT 1 CHECK (active IN (0,1)),
  created_at      INTEGER NOT NULL
);

CREATE TABLE coupon_redemptions (
  id         TEXT PRIMARY KEY,
  coupon_id  TEXT NOT NULL,
  user_id    TEXT NOT NULL,                          -- pseudonymised on deletion like passes
  pass_id    TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  UNIQUE (coupon_id, user_id)                        -- one redemption per user per coupon
);

CREATE TABLE plan_limits (                          -- editable without deploy (npm run set-limit / D1 console)
  limit_group TEXT NOT NULL,                         -- LimitGroup (Part 6.2)
  feature     TEXT NOT NULL,                         -- QuotaFeature | CountCap (Part 6.3)
  max_value   INTEGER NOT NULL CHECK (max_value >= 0),
  updated_at  INTEGER NOT NULL,
  PRIMARY KEY (limit_group, feature)
);

CREATE TABLE app_settings (                         -- editable without deploy (npm run set-setting / ai-off)
  key        TEXT PRIMARY KEY,                       -- SettingKey (Part 6.8)
  value      TEXT NOT NULL,                          -- JSON-encoded
  updated_at INTEGER NOT NULL,
  updated_by TEXT NOT NULL                           -- 'migration' | 'script:<name>' | 'console'
);

CREATE TABLE quota_usage (
  user_id    TEXT NOT NULL,
  feature    TEXT NOT NULL,                          -- QuotaFeature
  period_key TEXT NOT NULL,                          -- 'f:<n>' | 'p:<passId>' | 'l:<YYYY-MM>' (Part 6.4)
  used       INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (user_id, feature, period_key)
);

CREATE TABLE ai_budget_daily (                      -- global daily USD guard (all AI) + anonymous sub-budget
  day                     TEXT PRIMARY KEY,          -- 'YYYY-MM-DD' UTC
  reserved_micro_usd      INTEGER NOT NULL DEFAULT 0,-- in-flight worst-case estimates
  spent_micro_usd         INTEGER NOT NULL DEFAULT 0,-- settled actual cost (includes anon)
  anon_reserved_micro_usd INTEGER NOT NULL DEFAULT 0,
  anon_spent_micro_usd    INTEGER NOT NULL DEFAULT 0,
  calls                   INTEGER NOT NULL DEFAULT 0,
  anon_calls              INTEGER NOT NULL DEFAULT 0,
  updated_at              INTEGER NOT NULL
);

CREATE TABLE llm_calls (                            -- never stores prompts, outputs or resume text
  id                  TEXT PRIMARY KEY,
  user_id             TEXT,                          -- NULL for anonymous checker; set NULL on account deletion
  anon                INTEGER NOT NULL CHECK (anon IN (0,1)),
  feature             TEXT NOT NULL,                 -- AiFeature (Part 6.5)
  model               TEXT NOT NULL,
  input_tokens        INTEGER NOT NULL,
  cached_input_tokens INTEGER NOT NULL,
  output_tokens       INTEGER NOT NULL,              -- includes reasoning tokens
  reasoning_tokens    INTEGER NOT NULL,
  cost_micro_usd      INTEGER NOT NULL,
  latency_ms          INTEGER NOT NULL,
  ok                  INTEGER NOT NULL CHECK (ok IN (0,1)),
  error_code          TEXT,                          -- ErrorCode or provider status class
  created_at          INTEGER NOT NULL
);
CREATE INDEX llm_calls_created ON llm_calls (created_at);
CREATE INDEX llm_calls_user ON llm_calls (user_id, created_at);

CREATE TABLE ip_caps (                              -- raw IPs never stored; rows purged after 2 days
  day     TEXT NOT NULL,
  ip_hash TEXT NOT NULL,                             -- first 32 hex of HMAC-SHA256(IP_HASH_SECRET, day + ':' + ip)
  bucket  TEXT NOT NULL CHECK (bucket IN ('checker_ai','events_public')),
  count   INTEGER NOT NULL,
  PRIMARY KEY (day, ip_hash, bucket)
);

CREATE TABLE events (                               -- first-party funnel events (Part 6.7)
  id           TEXT PRIMARY KEY,
  name         TEXT NOT NULL,                        -- EventName
  user_id      TEXT,                                 -- NULL for anonymous events; set NULL on account deletion
  props        TEXT,                                 -- JSON, keys allowlisted per event
  country      TEXT,
  campus_code  TEXT,                                 -- copied from users at write time
  utm_source   TEXT,                                 -- copied from users.ft_* at write time
  utm_medium   TEXT,
  utm_campaign TEXT,
  day          TEXT NOT NULL,                        -- 'YYYY-MM-DD' UTC
  created_at   INTEGER NOT NULL
);
CREATE INDEX events_name_day ON events (name, day);
CREATE INDEX events_user_name ON events (user_id, name);

CREATE TABLE account_deletions (                    -- audit of deletions; holds no personal data once completed
  id                   TEXT PRIMARY KEY,
  clerk_user_id        TEXT,                         -- kept only until Clerk deletion succeeds, then set NULL
  status               TEXT NOT NULL CHECK (status IN ('started','clerk_pending','completed')),
  docs_rows            INTEGER NOT NULL DEFAULT 0,
  core_rows            INTEGER NOT NULL DEFAULT 0,
  passes_pseudonymised INTEGER NOT NULL DEFAULT 0,
  requested_at         INTEGER NOT NULL,
  completed_at         INTEGER
);
```

### 2.3 Docs database (`DB_DOCS`)

```sql
CREATE TABLE resumes (                              -- master resumes
  id             TEXT PRIMARY KEY,
  user_id        TEXT NOT NULL,
  title          TEXT NOT NULL,                      -- 1-80 chars, user-visible name
  source         TEXT NOT NULL CHECK (source IN ('blank','import','duplicate','sample')),
  data           TEXT NOT NULL,                      -- ResumeData JSON (Part 7), max 65,536 bytes
  schema_version INTEGER NOT NULL,
  revision       INTEGER NOT NULL,                   -- starts at 1, +1 per save (optimistic concurrency)
  size_bytes     INTEGER NOT NULL,
  created_at     INTEGER NOT NULL,
  updated_at     INTEGER NOT NULL
);
CREATE INDEX resumes_user_updated ON resumes (user_id, updated_at);

CREATE TABLE tailored_versions (                    -- one per job; a full ResumeData copy of its master + the JD
  id               TEXT PRIMARY KEY,
  user_id          TEXT NOT NULL,
  resume_id        TEXT NOT NULL,                    -- master resumes.id
  title            TEXT NOT NULL,
  job_title        TEXT NOT NULL,                    -- 0-120
  company          TEXT NOT NULL,                    -- 0-120
  jd_text          TEXT NOT NULL,                    -- 1-15,000 chars
  jd_hash          TEXT NOT NULL,                    -- sha256 hex of normalized jd_text
  keywords         TEXT,                             -- JdKeywords JSON (Part 7.3), cached LLM extraction
  keywords_jd_hash TEXT,                             -- jd_hash the cache was computed for; mismatch = stale
  keywords_model   TEXT,
  match_percent    INTEGER,                          -- 0-100, recomputed on the server at every save when keywords exist
  data             TEXT NOT NULL,                    -- ResumeData JSON
  schema_version   INTEGER NOT NULL,
  revision         INTEGER NOT NULL,
  size_bytes       INTEGER NOT NULL,
  created_at       INTEGER NOT NULL,
  updated_at       INTEGER NOT NULL
);
CREATE INDEX tailored_user_resume ON tailored_versions (user_id, resume_id);
CREATE INDEX tailored_user_updated ON tailored_versions (user_id, updated_at);

CREATE TABLE cover_letters (
  id           TEXT PRIMARY KEY,
  user_id      TEXT NOT NULL,
  tailored_id  TEXT NOT NULL,                        -- tailored_versions.id
  title        TEXT NOT NULL,
  body         TEXT NOT NULL,                        -- plain text, paragraphs separated by '\n\n', max 6,000 chars
  revision     INTEGER NOT NULL,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL
);
CREATE INDEX cover_letters_user_tailored ON cover_letters (user_id, tailored_id);
```

**Cascades (enforced in `services/documents.ts`).** Deleting a master deletes its tailored versions and their cover letters. Deleting a tailored version deletes its cover letters.

**Account deletion (`data/account-data.ts`):**

| Data | What happens |
|---|---|
| docs | All rows for the user are deleted. |
| `consents`, `quota_usage`, `checkout_intents`, `users` | Rows are deleted. |
| `events`, `llm_calls` | `user_id` is set to NULL. |
| `passes`, `coupon_redemptions` | `user_id` is set to one fresh `'del_<uuidv7>'` value, and `passes.pseudonymised_at` is set. |
| Clerk | The Clerk user is deleted through the Backend API. |

The deletion-completeness test asserts that no row in any table still contains the user id, the Clerk id or the email.

---

## 3. Endpoints

The contract registry `API` in `src/shared/api/contract.ts` declares each endpoint once, with its key, method, path and the `params`, `query`, `body` and `output` zod schemas. The server registers it with `route(API.<key>, { access, ai?, handler })`. Schemas live in `src/shared/schemas/api.ts` and are named `<Key>Params`, `<Key>Query`, `<Key>Body` and `<Key>Output` in PascalCase (for example `ResumeSaveBody`). Ids in paths are UUIDv7. `contract-coverage.test.ts` fails unless every contract has exactly one route and every route has a contract.

| Key | Method | Path | Access | `ai` | Purpose |
|---|---|---|---|---|---|
| `health` | GET | `/api/health` | public | | Liveness check: `{ok, env, release, core, docs}` (one `SELECT 1` per DB). The smoke test uses it. |
| `meta` | GET | `/api/meta` | public | | Country, default currency (INR if `IN`), `aiEnabled`, `anonAiEnabled`, checkout enabled per currency, `lifetimeAvailable`, `readOnly`, `banner` |
| `checkerTips` | POST | `/api/checker/tips` | public | `checker_tips` | Turnstile-gated AI tips (top 5 fixes), plus JD keywords if a JD is given, for the no-signup checker. Body requires `aiConsent: true`. |
| `eventsPublic` | POST | `/api/events` | public | | Anonymous client beacons (`checker_used`, `checker_cta_clicked`, `pricing_viewed` only). IP-capped. |
| `webhookDodo` | POST | `/api/webhooks/dodo` | webhook | | Dodo Standard Webhooks: verify, dedupe, re-fetch, then grant or revoke |
| `webhookRazorpay` | POST | `/api/webhooks/razorpay` | webhook | | Razorpay `order.paid` / `payment.captured` / refund / dispute: verify, dedupe, re-fetch, then grant or revoke |
| `meInit` | POST | `/api/me/init` | user | | Idempotent: ensures the users row, sets first-touch attribution and campus once, emits `signup_completed` on creation, returns `Me` |
| `meGet` | GET | `/api/me` | user | | `Me`: profile, `AccessState`, quota snapshot, count caps, consents, `isAdmin`, flags |
| `meUpdate` | PATCH | `/api/me` | user | | Update `currency`; set `campusCode` only if currently NULL |
| `meConsent` | POST | `/api/me/consents` | user | | Record a grant or withdrawal for `terms`, `privacy` or `ai_processing` at the current version |
| `meEvent` | POST | `/api/me/events` | user | | Signed-in client beacon (`paywall_viewed` only) |
| `meExport` | GET | `/api/me/export` | user | | Full JSON export: core + docs rows + Clerk profile (name, email) |
| `meDelete` | DELETE | `/api/me` | user | | Delete the account (body `{confirm: "DELETE"}`): erase, pseudonymise, delete the Clerk user |
| `resumesList` | GET | `/api/resumes` | user | | Masters with nested tailored summaries (id, title, jobTitle, company, matchPercent, updatedAt) |
| `resumeCreate` | POST | `/api/resumes` | user | | Create a master (`source` blank/import/sample, optional `data`). Enforces `resumes_max`. |
| `resumeGet` | GET | `/api/resumes/:id` | user | | Full master, including `data` and `revision` |
| `resumeSave` | PUT | `/api/resumes/:id` | user | | Save `data` with `baseRevision`. Returns 409 `REVISION_CONFLICT` if stale. |
| `resumeUpdateMeta` | PATCH | `/api/resumes/:id` | user | | Rename (`title`) |
| `resumeDuplicate` | POST | `/api/resumes/:id/duplicate` | user | | Copy a master (`source` duplicate). Enforces `resumes_max`. |
| `resumeDelete` | DELETE | `/api/resumes/:id` | user | | Delete a master, its tailored versions and their cover letters |
| `tailoredCreate` | POST | `/api/tailored` | user | | Create from a master + JD (`resumeId`, `jobTitle`, `company`, `jdText`). Copies the master data. Enforces `tailored_max`. |
| `tailoredGet` | GET | `/api/tailored/:id` | user | | Full tailored version, including JD, cached keywords and cover letter summaries |
| `tailoredSave` | PUT | `/api/tailored/:id` | user | | Save `data` with `baseRevision`. The server recomputes `match_percent`. |
| `tailoredUpdateMeta` | PATCH | `/api/tailored/:id` | user | | Edit `title`, `jobTitle`, `company` or `jdText`. A JD change makes the keyword cache stale. |
| `tailoredDelete` | DELETE | `/api/tailored/:id` | user | | Delete a tailored version and its cover letters |
| `coverLetterCreate` | POST | `/api/cover-letters` | paid | | Save an accepted cover letter for a tailored version. Enforces `cover_letters_max`. |
| `coverLetterGet` | GET | `/api/cover-letters/:id` | user | | Read one cover letter (still allowed after the pass expires) |
| `coverLetterSave` | PUT | `/api/cover-letters/:id` | user | | Save an edited `body`/`title` with `baseRevision` |
| `coverLetterDelete` | DELETE | `/api/cover-letters/:id` | user | | Delete a cover letter |
| `aiImport` | POST | `/api/ai/import` | user | `import_map` | Map extracted text (max 20,000 chars) to a `ResumeData` draft with per-section confidence and honesty flags. Nothing is saved. |
| `aiJdKeywords` | POST | `/api/ai/jd-keywords` | user | `jd_keywords` | Extract JD keywords for a tailored version and cache them on its row. On a cache hit it returns the cache without consuming quota. |
| `aiBullet` | POST | `/api/ai/bullet` | user | `bullet_rewrite` | Up to 3 rewrites of one bullet (optionally toward JD keywords), each with honesty flags |
| `aiSummary` | POST | `/api/ai/summary` | user | `summary` | Up to 3 summary candidates from the document, each with honesty flags |
| `aiCoverLetter` | POST | `/api/ai/cover-letter` | paid | `cover_letter` | Cover letter draft for a tailored version, with honesty flags. Nothing is saved. |
| `exportAuthorize` | POST | `/api/exports` | user | | Soft export gate: consumes `pdf_export`/`docx_export` quota and writes `first_export` (once) and `export_used` |
| `checkout` | POST | `/api/billing/checkout` | user | | Create an intent from the server catalog (+ coupon for INR). Returns a Dodo `checkoutUrl`, or a Razorpay `{orderId, keyId, amountMinor, currency, intentId}`. |
| `razorpayVerify` | POST | `/api/billing/razorpay/verify` | user | | Client callback: HMAC(order_id\|payment_id) check, re-fetch the payment, then the same idempotent `grantPass()` the webhook uses |
| `intentGet` | GET | `/api/billing/intents/:id` | user | | Intent status (`created\|paid\|failed\|expired`) for the return page to poll |
| `couponPreview` | POST | `/api/billing/coupons/preview` | user | | Validate a code for a plan and return the price after the discount |
| `couponRedeem` | POST | `/api/billing/coupons/redeem` | user | | Redeem a 100%-off code: `grantPass` with provider `coupon`, no checkout |
| `passesList` | GET | `/api/billing/passes` | user | | The caller's passes (receipt list, with revoked status) |
| `adminOverview` | GET | `/api/admin/overview` | admin | | KPIs for today / 7 d / 30 d: signups, paid passes and revenue per currency, LLM spend vs budget |
| `adminFunnel` | GET | `/api/admin/funnel` | admin | | Funnel counts by `day \| country \| campus` for `from`/`to` |
| `adminCosts` | GET | `/api/admin/costs` | admin | | LLM spend by day, feature and model; anonymous vs signed-in; budget headroom |
| `adminLimits` | GET | `/api/admin/limits` | admin | | Current `plan_limits` and `app_settings`, quota headroom distribution, D1 writes/day estimate |
| `adminCoupons` | GET | `/api/admin/coupons` | admin | | Redemptions per coupon code and college |

**Access semantics (enforced by `route()`)**

| Access | Checks before the handler runs | What the handler receives |
|---|---|---|
| `public` | CSRF for non-GET. The Turnstile + IP cap + anonymous budget chain when `ai` is set. | `req` (ReqMeta), `svc: {health, meta, checker, events}` |
| `user` | Clerk JWT → `ensureUser` → `AccessState`. CSRF for non-GET. `read_only` blocks non-GET. Terms + privacy consent at the current version is required for non-GET routes outside `/api/me*`. | `req`, `user`, `access`, `repo` (scoped), `svc` (user services), `llm` (only if `ai` is set) |
| `paid` | Everything `user` checks, plus `access.tier !== 'free'`, else `PAYWALL`. | same as `user` |
| `webhook` | Raw body ≤ 64 KB. No auth or CSRF. Signature verification happens inside `ctx.billing`. | `req`, `rawBody`, `headers`, `billing: WebhookBilling` |
| `admin` | Clerk JWT, and the verified email must be in `ADMIN_EMAILS`, else `FORBIDDEN`. | `req`, `admin: {email}`, `reports: AdminReports` (aggregates only) |

Handlers never receive `env`, a DB binding or a Drizzle instance.

---

## 4. Pages and routes

### 4.1 Astro static pages (`src/pages`, pre-rendered, served free as static assets)

| Path | File | Content source | Islands | In sitemap |
|---|---|---|---|---|
| `/` | `index.astro` | inline copy | `PricingTable` (teaser) | yes |
| `/pricing` | `pricing.astro` | `PLANS` from `shared/config/plans.ts` | `PricingTable`, `CurrencySwitch` | yes |
| `/resume-checker` | `resume-checker.astro` | inline copy | `CheckerApp` (`client:only="react"`) | yes |
| `/how-match-score-works` | `how-match-score-works.astro` | formula from `shared/keywords/match.ts` docs | none | yes |
| `/templates` | `templates.astro` | `TEMPLATES` from `shared/config/templates.ts` + preview PNGs | none | yes |
| `/campus` | `campus.astro` | inline copy (placement-cell pilot and cohort plan) | none | yes |
| `/about` | `about.astro` | inline copy | none | yes |
| `/contact` | `contact.astro` | `support@<domain>` (Email Routing) | none | yes |
| `/legal/terms`, `/legal/privacy`, `/legal/refunds` | `legal/[slug].astro` | collection `legal` | none | yes |
| `/legal/licenses` | `legal/licenses.astro` | `src/generated/licenses.json` | none | yes |
| `/guides` | `guides/index.astro` | collection `guides` (non-draft) | none | yes |
| `/guides/<slug>` | `guides/[slug].astro` | collection `guides` | none | yes |
| `/app`, `/app/*` (Part 4.2) | `app/[...slug].astro` | `getStaticPaths` from `APP_ROUTES`; renders `<App client:only="react" />` in `AppShellLayout` | `App` | no (noindex) |
| `/robots.txt` | `robots.txt.ts` | `Disallow: /` unless `PUBLIC_APP_ENV === 'production'`; always disallows `/app` | n/a | n/a |
| `/404` | `404.astro` | inline | none | no |

**Content collections (`src/content.config.ts`)**

```ts
guides: { title: string /*≤70*/; description: string /*≤160*/; publishedAt: Date; updatedAt: Date;
          market: 'india' | 'global' | 'both'; primaryKeyword: string; draft: boolean }   // slug = file name
legal:  { title: string; version: string /* 'YYYY-MM-DD', must equal LEGAL_VERSIONS */; effectiveDate: Date }
```

Initial guide slugs: `fresher-resume-format-campus-placements`, `how-to-read-a-job-description-for-keywords`, `resume-bullet-points-with-metrics`, `cgpa-on-resume-india`, `one-page-resume-for-freshers`.

### 4.2 SPA routes (`src/app/routes`, React Router; paths in `APP_ROUTES`)

Ids travel in **query strings**, never in path segments, so every SPA URL is a pre-built static file. Clerk components use `routing="hash"`.

| Path | Component | Guard | Query | Purpose |
|---|---|---|---|---|
| `/app` | `DashboardPage` | onboarded | none | Masters and their tailored versions; create, import, duplicate, delete; quota meters; banner |
| `/app/sign-in` | `SignInPage` | none | `redirect` | Clerk `<SignIn>`: Google, email code, email link |
| `/app/sign-up` | `SignUpPage` | none | `redirect`, `campus` | Clerk `<SignUp>` |
| `/app/onboarding` | `OnboardingPage` | signed in | `next` | `POST /api/me/init` → terms + privacy consent (required) → AI consent (optional, explained) → currency → campus code |
| `/app/import` | `ImportPage` | onboarded | none | Upload PDF/DOCX or paste text → browser extraction → `aiImport` → confirm section by section → `resumeCreate` |
| `/app/editor` | `EditorPage` | onboarded | `resume=<id>` **or** `tailored=<id>` | Section forms, drag reorder, live PDF preview, checks, AI suggestions with diff and flags, template picker, export dialog; JD/match panel when `tailored` |
| `/app/tailor/new` | `TailorNewPage` | onboarded | `resume=<id>` | Paste JD + job title + company → `tailoredCreate` → `aiJdKeywords` → editor |
| `/app/cover-letter` | `CoverLetterPage` | onboarded (generating needs a pass) | `tailored=<id>`, optional `id=<coverLetterId>` | Generate (paid), edit, save, export a cover letter |
| `/app/billing` | `BillingPage` | onboarded | optional `plan`, `coupon` | Plans for the currency, coupon field, checkout (Dodo redirect or Razorpay modal), my passes |
| `/app/billing/return` | `BillingReturnPage` | onboarded | `intent=<id>` | Dodo return URL: polls `intentGet` every 2 s for up to 60 s, then shows "payment received, activating" guidance |
| `/app/settings` | `SettingsPage` | onboarded | none | Currency, consents (withdraw AI), download export, delete account |
| `/app/admin` | `AdminPage` | admin (`me.isAdmin`) | `from`, `to` | Read-only funnel, costs, limits, coupons |

"Onboarded" means `Me.consents.terms` and `Me.consents.privacy` equal the current `LEGAL_VERSIONS`. Any unknown `/app/*` path gets the static 404 page.

---

## 5. Modules

### 5.1 `src/worker.ts` and `src/server`

| Module | Single responsibility | Exports |
|---|---|---|
| `worker.ts` | Worker entry, wrapped with Sentry | `default { fetch, scheduled }` (`withSentry(sentryOptions, …)`) |
| `server/app.ts` | Build the Hono app from `ROUTES` with the global error and not-found handlers | `createApp(): Hono<{ Bindings: Env }>` |
| `server/env.ts` | Env typing and validation of vars | `type Env`, `type PublicConfig`, `parseEnv(env): ValidatedEnv` |
| `core/route.ts` | The only way to declare an endpoint; enforces access, CSRF, body limits, read-only, consent, zod I/O, AI guard | `route(contract, def)`, `type Access`, `type RouteDef`, `type HandlerCtx<A>`, `mountRoutes(app, routes)` |
| `core/auth.ts` | All Clerk usage (the replaceable auth seam) | `verifySession(req, env): Promise<AuthIdentity \| null>` (`@clerk/backend` `verifyToken` with `jwtKey` + `authorizedParties`), `getAuthProfile(env, clerkUserId)`, `deleteAuthUser(env, clerkUserId)`, `isAdminEmail(env, email)` |
| `core/http.ts` | Envelope, errors, request guards | `class ApiError`, `ok(data)`, `errorResponse(err)`, `assertSameOrigin(req, origin)`, `assertJson(req)`, `readJsonBody(req, maxBytes)`, `readRawBody(req, maxBytes)` |
| `core/context.ts` | Context types per access level | `ReqMeta`, `UserIdentity`, `PublicCtx`, `UserCtx`, `PaidCtx`, `WebhookCtx`, `AdminCtx` |
| `core/logger.ts` | Structured logs with redaction; the only `console.*` user | `log.info/warn/error(event, fields)`, `userHash(env, userId)` |
| `core/sentry.ts` | Sentry options and scrubbing | `sentryOptions(env)`, `scrubEvent(event)` (drops request bodies, breadcrumbs, resume fields) |
| `core/crypto.ts` | WebCrypto helpers | `sha256Hex(s)`, `hmacSha256Hex(key, msg)`, `hmacSha256Base64(key, msg)`, `timingSafeEqual(a, b)` |
| `core/ids.ts` | Id generation | `newId(): string` (UUIDv7), `pseudonymId(): string` (`'del_' + uuidv7`) |
| `core/clock.ts` | Time source (injectable in tests) | `now()`, `utcDay(ms)`, `utcMonth(ms)`, `startOfNextUtcMonth(ms)` |
| `data/db.ts` | Drizzle clients (internal to `data/`) | `coreDb(env)`, `docsDb(env)` |
| `data/schema/core.ts`, `data/schema/docs.ts` | Drizzle table definitions matching Part 2 | one table object per table |
| `data/scoped.ts` | **User-scoped repository: every statement has `user_id = boundUserId`** | `scoped(env, userId): UserRepo` with `profile`, `consents`, `passes` (read-only), `intents`, `quotas`, `resumes`, `tailored`, `coverLetters` |
| `data/system.ts` | Non-user repositories | `systemRepo(env): SystemRepo` with `users.ensure(clerkUserId, email, country)`, `settings.getAll()`, `planLimits.forGroup()`, `budget.reserve/settle()`, `ipCaps.hit()`, `llmCalls.insert()`, `events.insert()`, `webhookEvents.claim/setOutcome()`, `intents.getById/markStatus()`, `coupons.findByCode/incrementIfAvailable()`, `redemptions.insert()`, `passes.lifetimeCount()`, `health.ping()` |
| `data/ledger.ts` | The `passes` writes (import allowed only from `billing/grant.ts`) | `insertPass(row)`, `markPassRevoked(id, reason, at)`, `findPassByProviderPaymentId(id)` |
| `data/admin.ts` | Aggregate-only queries (returns numbers, never rows with user content) | `adminRepo(env): AdminRepo` |
| `data/account-data.ts` | Export and erasure across both DBs for one user | `exportUserData(env, userId)`, `eraseUserData(env, userId): ErasureCounts` |
| `services/index.ts` | Wiring of services per access level | `buildPublicServices(env, req)`, `buildUserServices(env, user, repo, access)`, `buildAdminReports(env)`, `buildWebhookBilling(env)` |
| `services/account.ts` | Me, consents, export, deletion | `initMe`, `getMe`, `updateMe`, `recordConsent`, `exportAccount`, `deleteAccount` |
| `services/access.ts` | The caller's entitlement state | `loadAccess(repo, user, now): AccessState` (calls `computeAccess`) |
| `services/quota.ts` | Period quotas and count caps | `reserve(feature): Reservation`, `refund(reservation)`, `snapshot(): QuotaSnapshot`, `assertCountCap(cap, currentCount)` |
| `services/settings.ts` | Typed settings and limits, memoised per request | `getSettings(sys): Settings`, `getLimits(sys, group): LimitValues` |
| `services/documents.ts` | Resume, tailored and cover letter rules (caps, revisions, cascades, match recompute) | `listDocuments`, `createResume`, `getResume`, `saveResume`, `renameResume`, `duplicateResume`, `deleteResume`, `createTailored`, `getTailored`, `saveTailored`, `updateTailoredMeta`, `deleteTailored`, `createCoverLetter`, `getCoverLetter`, `saveCoverLetter`, `deleteCoverLetter` |
| `services/exports.ts` | Soft export gate | `authorizeExport({docKind, docId, format, templateId}): ExportAuthorizeOutput` |
| `services/events.ts` | First-party events with allowlisted props | `track(name, props)`, `trackAnonymous(name, props, req)` |
| `services/abuse.ts` | IP hashing, IP caps, Turnstile | `ipHash(env, ip, day)`, `hitIpCap(bucket, limit)`, `verifyTurnstile(env, token, ip, expectedAction)` |
| `services/checker.ts` | Anonymous checker AI flow | `checkerTips(input, req): CheckerTipsOutput` |
| `services/meta.ts` | Public meta and health | `getMeta(req)`, `getHealth()` |
| `services/admin.ts` | Admin reports | `overview`, `funnel`, `costs`, `limits`, `coupons` |
| `jobs/daily.ts` | Scheduled maintenance | `runDaily(env)`: purge `ip_caps` older than 2 days; mark `created` intents older than 48 h `expired`; delete `webhook_events` older than 90 days; delete `events` and `llm_calls` older than 400 days; retry `account_deletions` in `clerk_pending` |
| `billing/provider.ts` | Provider abstraction | `interface PaymentProvider`, `type VerifiedPaymentEvent`, `type ProviderPayment` |
| `billing/products.ts` | Plan ↔ Dodo product id mapping per env | `dodoProductId(env, planId)`, `planIdForDodoProduct(env, productId)` |
| `billing/dodo.ts` | Dodo REST + Standard Webhooks | `createDodoCheckout`, `verifyDodoWebhook`, `parseDodoEvent`, `fetchDodoPayment` |
| `billing/razorpay.ts` | Razorpay REST + HMAC | `createRazorpayOrder`, `verifyRazorpayPaymentSignature`, `verifyRazorpayWebhook`, `parseRazorpayEvent`, `fetchRazorpayPayment` |
| `billing/checkout.ts` | Checkout orchestration | `startCheckout(user, {planId, couponCode}): CheckoutOutput`, `verifyRazorpayCallback(user, body)` |
| `billing/coupons.ts` | Our coupons | `normalizeCouponCode`, `previewCoupon`, `priceWithCoupon`, `redeemFreeCoupon` |
| `billing/grant.ts` | **The only writer of `passes`** | `grantPass(input: GrantInput): GrantResult` (idempotent on `provider_payment_id`), `revokePass(input: RevokeInput)` |
| `billing/webhooks.ts` | Webhook flows | `handleDodoWebhook(rawBody, headers)`, `handleRazorpayWebhook(rawBody, headers)` (exposed to handlers as `WebhookBilling`) |
| `ai/llm-config.ts` | **The only place with model ids, token prices and per-feature settings** | `MODELS`, `MODEL_PRICES_MICRO_USD_PER_MTOK`, `AI_FEATURE_CONFIG`, `modelFor(feature, tier)` |
| `ai/provider.ts` | LLM abstraction | `interface LlmProvider`, `getLlmProvider(env)` (`AI_PROVIDER` = `openai \| mock`) |
| `ai/openai.ts` | **The only code that calls OpenAI** (Responses API, `store:false`, strict JSON schema, no tools, no streaming) | `openAiProvider(env): LlmProvider` |
| `ai/mock.ts` | Deterministic local/test outputs | `mockProvider(): LlmProvider` |
| `ai/cost.ts` | Cost math | `costMicroUsd(model, usage)`, `estimateMaxCostMicroUsd(model, inputChars, maxOutputTokens)` |
| `ai/guard.ts` | The ordered AI pipeline used by `route()` | `createGuardedLlm(env, ctx, feature): GuardedLlm`, which runs: (1) consent (user) or Turnstile + IP cap (anon); (2) settings + limits; (3) atomic quota reservation + daily budget reservation; (4) model call; (5) zod re-validation; (6) `llm_calls` row + budget settle; (7) refund on provider or validation error |
| `ai/tasks/<feature>.ts` | One AI task: prompt + output schema + post-processing (honesty check) | `task: AiTask<In, Out>` = `{ feature, buildPrompt(input), outputSchema, postProcess(output, source) }` |
| `routes/index.ts` | The route manifest | `ROUTES: RouteDef[]` (used by `app.ts` and the generated tests) |
| `routes/{public,webhooks,me,resumes,tailored,cover-letters,ai,exports,billing,admin}.ts` | Route declarations for one group of contracts | arrays of `route(API.<key>, …)` |

`UserRepo` outline (in `data/scoped.ts`):

```ts
interface UserRepo {
  readonly userId: string;
  profile:      { get(): UserRow; update(p: Partial<Pick<UserRow,'currency'|'campusCode'>>): void;
                  setAttributionOnce(ft: FirstTouch): void; touchLastSeen(day: string): void };
  consents:     { latest(): ConsentState; record(kind: ConsentKind, version: string, granted: boolean): void };
  passes:       { list(): PassRow[] };                                   // read-only
  intents:      { create(row): void; get(id): IntentRow | null };
  quotas:       { reserve(feature, periodKey, limit): boolean; refund(feature, periodKey): void; usage(keys): UsageMap };
  resumes:      { list(); get(id); create(row); save(id, data, baseRevision); rename(id, title); delete(id); count() };
  tailored:     { listForUser(); get(id); create(row); save(id, data, baseRevision, matchPercent);
                  updateMeta(id, patch); setKeywords(id, kw, jdHash, model); deleteByResume(resumeId); delete(id); count() };
  coverLetters: { listByTailored(tailoredId); get(id); create(row); save(id, patch, baseRevision);
                  deleteByTailored(tailoredId); delete(id); count() };
}
```

### 5.2 `src/shared` (pure: imports nothing outside `src/shared` except npm libraries such as zod)

| Module | Single responsibility | Exports |
|---|---|---|
| `config/brand.ts` | Product identity placeholder | `BRAND_NAME = '[BRAND]'`, `BRAND_IS_PLACEHOLDER = true`, `SUPPORT_EMAIL_LOCAL_PART = 'support'` |
| `config/plans.ts` | Plan catalog (prices, durations, limit groups) | `PLAN_IDS`, `type PlanId`, `PLANS`, `getPlan(id)`, `plansForCurrency(c)`, `LIFETIME_PLAN_IDS` |
| `config/limits.ts` | Limit groups, quota features, count caps, seed values | `LIMIT_GROUPS`, `QUOTA_FEATURES`, `COUNT_CAPS`, `DEFAULT_PLAN_LIMITS` |
| `config/settings.ts` | Settings keys, schema, defaults | `SETTING_KEYS`, `SettingsSchema`, `DEFAULT_SETTINGS`, `type Settings` |
| `config/ai-features.ts` | AI feature ids and their quota keys (no model ids) | `AI_FEATURES`, `type AiFeature`, `QUOTA_KEY_FOR_FEATURE`, `PAID_ONLY_FEATURES` |
| `config/templates.ts` | Template ids and metadata | `TEMPLATE_IDS`, `type TemplateId`, `TEMPLATES` (name, family `'global' \| 'india'`, default section order, one-page target) |
| `config/routes.ts` | Static and SPA paths | `SITE_PATHS`, `APP_ROUTES` |
| `config/legal.ts` | Current legal versions | `LEGAL_VERSIONS: { terms, privacy, ai_processing }` |
| `config/input-limits.ts` | Every size cap | `MAX_RESUME_BYTES = 65536`, `MAX_IMPORT_CHARS = 20000`, `MAX_JD_CHARS = 15000`, `MAX_BULLET_CHARS = 400`, `MAX_SUMMARY_CHARS = 1200`, `MAX_COVER_LETTER_CHARS = 6000`, `MAX_TITLE_CHARS = 80` |
| `api/contract.ts` | Endpoint registry shared by client and server | `API`, `type ApiContract`, `type ApiKey` |
| `schemas/common.ts` | Primitive schemas | `UuidV7Schema`, `ItemIdSchema`, `CurrencySchema`, `YearMonthSchema`, `DayKeySchema` |
| `schemas/resume.ts` | Resume JSON (Part 7) | `ResumeDataSchema`, `type ResumeData`, `RESUME_SCHEMA_VERSION`, `emptyResume(templateId)`, `migrateResume(raw)`, `FieldRefSchema` |
| `schemas/api.ts` | Request/response schemas for every contract | `<Key>Params/Query/Body/Output`, `MeSchema`, `AccessStateSchema`, `QuotaSnapshotSchema` |
| `schemas/ai.ts` | LLM structured output schemas (strict) | `ImportMapOutput`, `JdKeywordsOutput`, `BulletRewriteOutput`, `SummaryOutput`, `CoverLetterOutput`, `CheckerTipsOutput` |
| `schemas/events.ts` | Event names and prop allowlists | `EVENT_NAMES`, `PUBLIC_CLIENT_EVENTS`, `USER_CLIENT_EVENTS`, `EVENT_PROPS` |
| `schemas/errors.ts` | Error codes | `ERROR_CODES`, `ERROR_STATUS`, `ApiErrorBodySchema` |
| `access/compute-access.ts` | Pure entitlement math | `computeAccess(passes: PassLike[], now: number, userCreatedAt: number): AccessState`, `freePeriod(now, createdAt)`, `passSegments(passes)` |
| `honesty/*` | Unsupported-claim detection | `buildHonestySource(resume: ResumeData, extraTexts: string[]): HonestySource`, `findUnsupportedClaims(text: string, source): HonestyFlag[]`, `flagResumeDraft(draft: ResumeData, source): FlaggedField[]` |
| `keywords/*` | JD match math | `normalizeTerm(s)`, `computeMatch(kw: JdKeywords, resume: ResumeData): MatchResult`, `extractKeywordsHeuristic(jdText): JdKeywords`, `MATCH_WEIGHTS = { required: 2, preferred: 1 }` |
| `rules/*` | Deterministic content checks | `runContentChecks(resume, opts: { pageCount?: number }): CheckResult[]`, `RULE_IDS` |
| `dictionaries/*` | Word lists | `SKILL_TERMS`, `DEGREE_PATTERNS`, `STRONG_VERBS`, `WEAK_VERBS`, `ORG_SUFFIXES`, `FIRST_PERSON_PRONOUNS` |
| `text/normalize.ts` | Text normalization | `normalizeWhitespace`, `capText(s, max)`, `stripControlChars` |
| `text/resume-text.ts` | Flatten a resume to plain text | `resumeToPlainText(data)`, `iterateBullets(data): Iterable<{ref: FieldRef, text}>` |
| `samples/sample-resume.ts` | Fictional sample | `SAMPLE_RESUME_GLOBAL`, `SAMPLE_RESUME_INDIA` |
| `money.ts` | Money formatting | `formatMoney(minor, currency, locale)`, `CURRENCY_SYMBOL` |
| `dates.ts` | Resume date helpers | `parseYearMonth`, `compareYearMonth`, `formatDateRange(range, fmt, locale)` |

**Match formula (canonical, published on `/how-match-score-works`):**
`percent = round(100 × (2 × matchedRequired + matchedPreferred) / (2 × totalRequired + totalPreferred))`. With zero keywords, `percent = null`. A keyword matches when its normalized term or any normalized alias appears as a whole token sequence in `resumeToPlainText(resume)`.

**Content check rule ids (`RULE_IDS`):** `missing_contact`, `missing_dates`, `date_order`, `weak_verb`, `no_metric`, `first_person`, `bullet_too_long`, `too_many_bullets`, `duplicate_bullet`, `tense_mismatch`, `summary_length`, `page_overflow`. Each result is `{ ruleId, severity: 'error' | 'warning' | 'tip', ref: FieldRef | null, message }`.

### 5.3 Browser-only modules (names fixed, details in their sections)

- `src/documents`: `registerPdfFonts`, `TEMPLATE_COMPONENTS`, `renderResumePdf`, `renderCoverLetterPdf`, `countPages`, `renderResumeDocx`, `renderCoverLetterDocx`, `drawPdf`, `extractText`, `extractPdfText`, `extractDocxText`, `saveBlob`, `exportFileName`.
- `src/app/lib`: `call` (api.ts), `useSessionToken`, `signOutAndClearDrafts`, `getDraft`/`putDraft`/`deleteDraft`/`clearAllDrafts`, `useAutosave`, `trackPublic`/`trackUser`, `captureFirstTouch`/`readFirstTouch`.
- `src/islands`: `CheckerApp`, `PricingTable`, `CurrencySwitch`.

---

## 6. Canonical ids and keys

### 6.1 Plan catalog (`src/shared/config/plans.ts`)

| `PlanId` | Display name | Currency | `amountMinor` | `durationDays` | `limitGroup` | Provider |
|---|---|---|---|---|---|---|
| `inr_7d` | 7-Day Pass | INR | 9900 | 7 | `pass_7d` | razorpay |
| `inr_30d` | Job Search Pass | INR | 19900 | 30 | `pass_30d` | razorpay |
| `inr_90d` | Placement Season Pass | INR | 49900 | 90 | `pass_90d` | razorpay |
| `inr_lifetime` | Lifetime | INR | 99900 | null | `lifetime` | razorpay |
| `usd_30d` | Job Search Pass | USD | 900 | 30 | `pass_30d` | dodo |
| `usd_90d` | Season Pass | USD | 1900 | 90 | `pass_90d` | dodo |
| `usd_lifetime` | Lifetime | USD | 4900 | null | `lifetime` | dodo |

- INR prices are shown as GST-inclusive.
- Lifetime plans are sold until the count of non-revoked lifetime passes (both currencies together) reaches `lifetime_founder_limit`.
- Campus cohorts and beta testers get INR plans through 100%-off coupons (`couponRedeem`). There is no other grant path.

### 6.2 Limit groups

`LimitGroup = 'free' | 'pass_7d' | 'pass_30d' | 'pass_90d' | 'lifetime'`.

### 6.3 Quota features, count caps and seed values (`DEFAULT_PLAN_LIMITS`, `migrations/core/0001_seed_defaults.sql`)

| Key | Kind | free | pass_7d | pass_30d | pass_90d | lifetime (per UTC month) |
|---|---|---|---|---|---|---|
| `ai_action` | quota (bullet_rewrite, summary, cover_letter) | 5 | 60 | 200 | 600 | 150 |
| `ai_import` | quota | 1 | 5 | 10 | 20 | 10 |
| `jd_analysis` | quota | 1 | 20 | 60 | 150 | 50 |
| `pdf_export` | quota (soft gate) | 1 | 100 | 300 | 900 | 300 |
| `docx_export` | quota (soft gate) | 0 | 100 | 300 | 900 | 300 |
| `resumes_max` | count cap (current rows) | 2 | 10 | 10 | 20 | 20 |
| `tailored_max` | count cap | 1 | 20 | 50 | 100 | 100 |
| `cover_letters_max` | count cap | 0 | 20 | 50 | 100 | 100 |

- `QuotaFeature = 'ai_action' | 'ai_import' | 'jd_analysis' | 'pdf_export' | 'docx_export'`.
- `CountCap = 'resumes_max' | 'tailored_max' | 'cover_letters_max'`.
- When a limit is 0, the route returns `PAYWALL` (not `QUOTA_EXHAUSTED`).
- Count caps block only creation. After a downgrade, existing rows stay readable, editable and deletable.

### 6.4 Quota periods (`computeAccess`)

| Tier | Condition | `periodKey` | `periodEndsAt` |
|---|---|---|---|
| `pass` | A non-revoked time-limited pass segment covers `now`. Segments stack in `purchased_at` order: start = max(purchased_at, previous end), end = start + durationDays. | `p:<passId>` | segment end |
| `lifetime` | Otherwise, a non-revoked lifetime pass exists | `l:<YYYY-MM>` | start of the next UTC month |
| `free` | Otherwise | `f:<n>`, n = floor((now − users.created_at) / 30 d) | users.created_at + (n+1) × 30 d |

```ts
type AccessState = { tier: 'free' | 'pass' | 'lifetime'; limitGroup: LimitGroup; planId: PlanId | null;
  activePassId: string | null; expiresAt: number | null; periodKey: string; periodEndsAt: number; hasLifetime: boolean };
```

### 6.5 AI features (`src/shared/config/ai-features.ts` + `src/server/ai/llm-config.ts`)

| `AiFeature` | Endpoint key | Quota / guard | Model (free → paid) | `max_output_tokens` | Reasoning effort |
|---|---|---|---|---|---|
| `import_map` | `aiImport` | `ai_import` | nano → nano | 6000 | minimal |
| `jd_keywords` | `aiJdKeywords` | `jd_analysis` | nano → nano | 1500 | minimal |
| `bullet_rewrite` | `aiBullet` | `ai_action` | nano → mini | 1200 | minimal |
| `summary` | `aiSummary` | `ai_action` | nano → mini | 1200 | minimal |
| `cover_letter` | `aiCoverLetter` | `ai_action`, paid only | — → mini | 2500 | minimal |
| `checker_tips` | `checkerTips` | Turnstile + IP bucket `checker_ai` + anonymous budget | nano | 1500 | minimal |

- `MODELS = { nano: 'gpt-5-nano-2025-08-07', mini: 'gpt-5-mini-2025-08-07' }`.
- Prices, in micro-USD per 1M tokens (input / cached input / output): nano 50,000 / 5,000 / 400,000; mini 250,000 / 25,000 / 2,000,000.
- Request timeout: 45 s.

### 6.6 Environment variables and secrets

**Worker vars** (non-secret, in `wrangler.jsonc` per environment):

| Name | local | staging | production |
|---|---|---|---|
| `APP_ENV` | `local` | `staging` | `production` |
| `SITE_ORIGIN` | `http://localhost:4321` | `https://staging.<domain>` | `https://<domain>` |
| `AI_PROVIDER` | `mock` | `openai` | `openai` |
| `CLERK_PUBLISHABLE_KEY` | dev instance `pk_test_…` | dev instance `pk_test_…` | prod instance `pk_live_…` |
| `CLERK_AUTHORIZED_PARTIES` | comma list of origins (= `SITE_ORIGIN`) | same | same |
| `DODO_MODE` | `test` | `test` | `live` |
| `DODO_PRODUCT_USD_30D`, `DODO_PRODUCT_USD_90D`, `DODO_PRODUCT_USD_LIFETIME` | test product ids | test product ids | live product ids |
| `RAZORPAY_KEY_ID` | `rzp_test_…` | `rzp_test_…` | `rzp_live_…` |
| `RAZORPAY_MODE` | `test` | `test` | `live` |
| `SENTRY_DSN` | empty (disabled) | staging DSN | prod DSN |
| `RELEASE` | `dev` | git SHA (`--var RELEASE:<sha>`) | git SHA |

`parseEnv` asserts that `RAZORPAY_KEY_ID` starts with `rzp_live_` exactly when `RAZORPAY_MODE` is `live`.

**Worker secrets.** A human sets each one with `wrangler secret put <NAME> --env <env>`. Locally they come from the git-ignored `.dev.vars`.

| Name | Used by |
|---|---|
| `CLERK_SECRET_KEY` | `core/auth.ts` (Backend API: profile, user deletion) |
| `CLERK_JWT_KEY` | `core/auth.ts` (networkless JWT verification, PEM) |
| `OPENAI_API_KEY` | `ai/openai.ts` (separate OpenAI project per environment) |
| `DODO_API_KEY` | `billing/dodo.ts` |
| `DODO_WEBHOOK_SECRET` | `billing/dodo.ts` |
| `RAZORPAY_KEY_SECRET` | `billing/razorpay.ts` (API auth + callback HMAC) |
| `RAZORPAY_WEBHOOK_SECRET` | `billing/razorpay.ts` |
| `TURNSTILE_SECRET_KEY` | `services/abuse.ts` |
| `IP_HASH_SECRET` | `services/abuse.ts`, `core/logger.ts` (daily IP hash, user hash) |
| `ADMIN_EMAILS` | `core/auth.ts` (comma-separated, lowercased) |

**Build-time public vars** (Astro `import.meta.env`, baked into static files; never secrets):
`PUBLIC_APP_ENV`, `PUBLIC_SITE_ORIGIN`, `PUBLIC_CLERK_PUBLISHABLE_KEY`, `PUBLIC_TURNSTILE_SITE_KEY`, `PUBLIC_RAZORPAY_KEY_ID`, `PUBLIC_SENTRY_DSN`, `PUBLIC_CF_BEACON_TOKEN`, `PUBLIC_RELEASE`.

**GitHub Actions.** Everything is repository-level, because GitHub Free private repos cannot use Environments. Names are suffixed `_STAGING` or `_PROD` where the values differ.

| Name | Kind | Purpose |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | secret | wrangler deploy, migrations, Time Travel, d1 export (scoped: Workers Scripts edit, D1 edit) |
| `CLOUDFLARE_ACCOUNT_ID` | secret | wrangler |
| `SENTRY_AUTH_TOKEN` | secret | source map upload (the step is skipped when the secret is empty) |
| `CLERK_SECRET_KEY_STAGING` | secret | `@clerk/testing` tokens for `e2e-staging` |
| `BACKUP_AGE_RECIPIENT` | variable | age public key for nightly dumps |
| `PUBLIC_<NAME>_STAGING`, `PUBLIC_<NAME>_PROD` | variables | one per build-time public var above (except `PUBLIC_APP_ENV` and `PUBLIC_RELEASE`, which the workflow sets) |

### 6.7 Event names (`src/shared/schemas/events.ts`)

| Event | Written by | Props (allowlist) |
|---|---|---|
| `checker_used` | client → `eventsPublic` | `{ fileKind: 'pdf'\|'docx'\|'paste', hasJd: boolean, aiTips: boolean }` |
| `checker_cta_clicked` | client → `eventsPublic` | `{ cta: 'fix_in_editor'\|'pricing' }` |
| `pricing_viewed` | client → `eventsPublic` | `{ currency: 'INR'\|'USD' }` |
| `signup_completed` | server (`users.ensure` created a row) | `{}` |
| `resume_created` | server | `{ source: 'blank'\|'import'\|'duplicate'\|'sample' }` |
| `import_used` | server (`aiImport`) | `{ ok: boolean }` |
| `ai_used` | server (every guarded call) | `{ feature: AiFeature, model: string, ok: boolean, tier: Tier }` |
| `tailored_created` | server | `{}` |
| `paywall_viewed` | client → `meEvent` | `{ trigger: 'ai_quota'\|'import_quota'\|'jd_quota'\|'pdf_limit'\|'docx'\|'cover_letter'\|'resume_cap'\|'tailored_cap'\|'nav' }` |
| `first_export` | server (`exportAuthorize`, first time per user) | `{ format: 'pdf'\|'docx', templateId: TemplateId, tier: Tier }` |
| `export_used` | server (`exportAuthorize`, every time) | `{ format, templateId, tier }` |
| `checkout_started` | server (`checkout`) | `{ planId, provider, currency, hasCoupon: boolean }` |
| `payment_succeeded` | server (`grantPass`, provider dodo/razorpay) | `{ planId, provider, currency, amountMinor }` |
| `coupon_redeemed` | server (`grantPass` with a coupon) | `{ couponCode, planId, percentOff }` |
| `pass_revoked` | server (`revokePass`) | `{ planId, provider, reason }` |
| `account_deleted` | server (`deleteAccount`; `user_id` NULL) | `{ hadPass: boolean }` |

`first_export` is written by the server inside `exportAuthorize`. The browser calls that endpoint before every export, so it replaces the "browser beacon" wording in the ADR.

### 6.8 Settings keys (`app_settings`, `src/shared/config/settings.ts`)

| Key | Type | Default | Effect |
|---|---|---|---|
| `ai_enabled` | boolean | `true` | Global AI kill switch (`npm run ai-off`) |
| `ai_daily_budget_micro_usd` | int | `1500000` | Global daily AI cap ($1.50), including anonymous calls |
| `anon_ai_enabled` | boolean | `true` | Checker AI tips on/off (rule-based checker always works) |
| `anon_ai_daily_budget_micro_usd` | int | `250000` | Anonymous daily sub-cap ($0.25) |
| `anon_checker_ip_daily_limit` | int | `3` | AI checker calls per IP hash per day |
| `events_ip_daily_limit` | int | `300` | `eventsPublic` beacons per IP hash per day |
| `checkout_usd_enabled` | boolean | `false` | Dodo checkout on/off (switched on after KYC) |
| `checkout_inr_enabled` | boolean | `false` | Razorpay checkout on/off (switched on after KYC) |
| `lifetime_founder_limit` | int | `200` | Max non-revoked lifetime passes sold |
| `dodo_max_discount_percent` | int | `50` | Lowest accepted Dodo pre-tax amount = list × (100 − value) / 100 (allows Dodo discount codes) |
| `read_only` | boolean | `false` | Blocks non-GET user/paid routes (`READ_ONLY`) during restores |
| `banner_message` | string (≤200) | `""` | In-app banner text |

### 6.9 npm scripts (canonical names)

| Group | Scripts |
|---|---|
| Dev | `dev` (astro dev :4321 with a Vite proxy `/api` → :8787), `dev:api` (wrangler dev), `preview` (astro build + wrangler dev serving `dist`) |
| Build and quality | `build`, `typecheck` (`astro check && tsc --noEmit`), `lint`, `format` |
| Tests | `test` (unit), `test:int` (workers pool), `test:e2e`, `test:golden` |
| Database | `db:generate` (both drizzle configs), `db:migrate:local`, `db:seed:local` |
| Checks | `check:migrations`, `check:boundaries`, `check:bundle`, `check:brand`, `check` (all four) |
| Ops (humans only, `--env` required) | `ai-off`, `ai-on`, `set-limit`, `set-setting`, `funnel`, `costs`, `coupon:create`, `coupon:list`, `replay-payments`, `restore-test` |

---

## 7. Resume JSON schema (`src/shared/schemas/resume.ts`)

### 7.1 Outline

```ts
export const RESUME_SCHEMA_VERSION = 1;

type Id = string;                                   // ItemIdSchema: 1-40 chars
type YearMonth = string;                            // 'YYYY-MM' or 'YYYY'
type DateRange = { start: YearMonth | null; end: YearMonth | null; current: boolean };
type Bullet = { id: Id; text: string };             // text 0-400 chars

type BuiltInSectionKey = 'summary' | 'experience' | 'internships' | 'positions' | 'education' | 'projects'
                       | 'skills' | 'certifications' | 'achievements' | 'languages';
type SectionKey = BuiltInSectionKey | `custom:${Id}`;

type ResumeData = {
  schemaVersion: 1;
  settings: {
    templateId: TemplateId;                         // see 7.2
    pageSize: 'A4' | 'LETTER';                      // default A4 for en-IN/en-GB, LETTER for en-US
    locale: 'en-IN' | 'en-US' | 'en-GB';
    dateFormat: 'MMM YYYY' | 'MM/YYYY' | 'YYYY';
    fontScale: number;                              // 0.9-1.1, step 0.05, default 1
    margins: 'normal' | 'narrow';
    sectionOrder: SectionKey[];                     // every visible section exactly once
    hiddenSections: SectionKey[];
    sectionTitles: Partial<Record<SectionKey, string>>; // overrides, ≤40 chars each
  };
  basics: {
    fullName: string;                               // 0-80
    headline: string;                               // 0-120
    email: string;                                  // valid email or ''
    phone: string;                                  // 0-30
    location: { city: string; region: string; country: string };
    links: { id: Id; label: string; url: string }[];   // max 5 (https only)
  };
  summary: { text: string };                        // 0-1200
  experience:  ExperienceItem[];                    // max 30 each
  internships: ExperienceItem[];
  positions:   ExperienceItem[];                    // positions of responsibility (campus clubs, coordinators)
  education:   EducationItem[];                     // max 10
  projects:    ProjectItem[];                       // max 20
  skills:      SkillGroup[];                        // max 12
  certifications: CertificationItem[];              // max 30
  achievements:   AchievementItem[];                // max 30
  languages:      LanguageItem[];                   // max 10
  custom:         CustomSection[];                  // max 3
};

type ExperienceItem = {
  id: Id; organization: string; title: string; location: string;
  employmentType: 'full_time' | 'part_time' | 'contract' | 'internship' | 'freelance' | 'volunteer' | 'other' | null;
  dates: DateRange; bullets: Bullet[];              // max 12 bullets
};
type EducationItem = {
  id: Id; institution: string;
  level: 'secondary' | 'senior_secondary' | 'diploma' | 'bachelor' | 'master' | 'doctorate' | 'other';
  degree: string; field: string;
  board: string;                                    // e.g. CBSE / ICSE / State board for Class X/XII; '' otherwise
  location: string; dates: DateRange;
  score: { kind: 'cgpa' | 'gpa' | 'percentage' | 'grade'; value: string; outOf: string } | null; // strings keep "8.72", "10"
  highlights: Bullet[];                             // max 6
  coursework: string[];                             // max 12
};
type ProjectItem = { id: Id; name: string; role: string; url: string; dates: DateRange;
                     techStack: string[] /* max 20 */; bullets: Bullet[] /* max 8 */ };
type SkillGroup = { id: Id; label: string /* e.g. 'Languages', 'Tools' */; items: string[] /* max 40 */ };
type CertificationItem = { id: Id; name: string; issuer: string; date: YearMonth | null; credentialId: string; url: string };
type AchievementItem = { id: Id; title: string; date: YearMonth | null; description: string /* 0-300 */ };
type LanguageItem = { id: Id; name: string;
                      proficiency: 'native' | 'fluent' | 'professional' | 'intermediate' | 'basic' | null };
type CustomSection = { id: Id; title: string;
                       items: { id: Id; heading: string; subheading: string; dates: DateRange | null; bullets: Bullet[] }[] };
```

Schema rules:

- Strings default to `''`, never `undefined`. Arrays default to `[]`.
- No photo, date of birth, gender, marital status, religion or nationality fields. The schema rejects them because of bias, privacy and ATS concerns.
- The serialized size must be ≤ `MAX_RESUME_BYTES` (65,536).
- `migrateResume(raw)` upgrades any older `schemaVersion` and is the only parser used when reading `data` from the DB or IndexedDB.

### 7.2 Templates (`TEMPLATE_IDS`)

| `TemplateId` | Family | Default `sectionOrder` |
|---|---|---|
| `global_classic` | global | summary, experience, projects, education, skills, certifications, achievements |
| `global_modern` | global | summary, experience, projects, skills, education, certifications, achievements |
| `global_compact` | global | summary, experience, skills, projects, education, certifications |
| `india_fresher` | india (one page) | education, skills, projects, internships, positions, achievements, certifications |
| `india_tech_fresher` | india (one page) | skills, projects, internships, education, achievements, certifications |
| `india_campus_academic` | india (one page) | education (Class X/XII/degree with scores), internships, projects, positions, achievements, skills, certifications |

### 7.3 Addressing and AI-related shapes

```ts
type FieldRef =
  | { section: 'summary' }
  | { section: 'experience' | 'internships' | 'positions' | 'projects'; itemId: Id; bulletId: Id }
  | { section: 'education'; itemId: Id; bulletId: Id }                 // highlights
  | { section: 'custom'; customSectionId: Id; itemId: Id; bulletId: Id };

type HonestyFlag = { kind: 'number' | 'percent' | 'money' | 'organization' | 'degree' | 'skill' | 'date';
                     text: string; start: number; end: number };       // offsets into the AI text

type Keyword = { term: string; aliases: string[];
                 category: 'hard_skill' | 'tool' | 'certification' | 'domain' | 'soft_skill' };
type JdKeywords = { roleTitle: string; seniority: 'intern' | 'entry' | 'mid' | 'senior' | null;
                    required: Keyword[] /* max 25 */; preferred: Keyword[] /* max 25 */ };
type MatchResult = { percent: number | null; matchedRequired: string[]; missingRequired: string[];
                     matchedPreferred: string[]; missingPreferred: string[] };
```

The honesty source for every AI call is the **stored** master resume (for a tailored document, its `resume_id` master plus the tailored data), plus the request's own input text. For `import_map` it is the extracted import text. Before any AI action, the client flushes a pending save, so the stored copy is current.

---

## 8. Invariants (copied into `CLAUDE.md`) and how they are enforced

| # | Invariant | Enforced by |
|---|---|---|
| 1 | Every `/api` endpoint is a contract in `shared/api/contract.ts`, registered once with `route()` and an explicit `access` | `route()` types (access required), `contract-coverage.test.ts`, `route-manifest.test.ts` snapshot (any new public, webhook or admin route fails until the snapshot is approved) |
| 2 | Only `src/server/data/**` references `DB_CORE` / `DB_DOCS` or creates Drizzle clients. Handlers never receive `env`. | ESLint `no-restricted-imports` + `check:boundaries` grep |
| 3 | Every statement in `data/scoped.ts` and `data/account-data.ts` filters on the bound `user_id`, including UPDATE and DELETE | `idor-matrix.test.ts`, a second review pass on any diff to `src/server/data/**` |
| 4 | Only `billing/grant.ts` imports `data/ledger.ts`. Access is derived only by `computeAccess`. There is no mutable plan column. | ESLint rule + `check:boundaries`, fast-check property tests |
| 5 | Only `ai/openai.ts` contacts OpenAI, and only through `GuardedLlm` (consent → limits → reserve → call → validate → log → refund). Model ids and prices exist only in `llm-config.ts`. | `check:boundaries` (the `api.openai.com` string and model id strings), `ai.test.ts` (quota consumed and refunded; `llm_calls` row written) |
| 6 | Prices and amounts come only from `PLANS` + `coupons` on the server. Client-sent amounts are ignored. Passes are granted only after a verified webhook or Razorpay callback with a provider re-fetch (product or order, amount, currency, status, mode). Grants are idempotent on `provider_payment_id` and the webhook event id. | webhook integration tests (bad signature, replay, duplicate, wrong amount/currency/product, test-mode event in prod) |
| 7 | Migrations are additive only, and applied migration files are never edited | `check:migrations` in CI |
| 8 | No `console.*` outside the two logger modules. No resume, JD, prompt, email or raw IP in logs, Sentry or `llm_calls`. | ESLint `no-console`, `logger-redaction.test.ts`, Sentry `scrubEvent` test |
| 9 | `src/shared` imports nothing from other `src/*` folders. `src/app`, `src/islands`, `src/documents` and `src/pages` never import `src/server`. | ESLint import boundaries |
| 10 | The product name is always `BRAND_NAME`. "Rezi" never appears. A prod deploy is blocked while `BRAND_IS_PLACEHOLDER` is true. | `check:brand` (CI and `deploy-production.yml`) |
| 11 | Secrets exist only as Worker secrets or GitHub secrets that humans set. None are in code, chat, `wrangler.jsonc` or the browser bundle. | gitleaks, `check:bundle` |
| 12 | Every AI output is validated with zod again and honesty-checked on the server. Flags are returned, and nothing is applied without the user accepting it. | `ai.test.ts`, honesty unit tests |
| 13 | Any diff touching `src/server/data/**`, `core/route.ts`, `core/auth.ts` or `billing/**` gets a second review (`/code-review` and `/security-review`) before a prod release | `pull_request_template.md` checklist, `CLAUDE.md` |
| 14 | Tests are written first. CI must be green before staging and production deploys. Tests are never skipped or deleted to make CI pass. | `ci.yml` gates the deploy workflows |
| 15 | Ids are UUIDv7, times are ms epoch UTC, money is integer minor units, and LLM cost is integer micro-USD | `common.ts` schemas, unit tests |

---

## 9. Things to verify at build time (each with its fallback)

| Item | How to verify | Fallback if it fails |
|---|---|---|
| `run_worker_first: ["/api/*"]` with `not_found_handling: "404-page"` sends only `/api/*` to the Worker | Deploy to staging. `/pricing` and `/app/editor` must not appear in Worker invocation metrics, and `/api/health` must return JSON. | Set `not_found_handling: "none"` and have the Worker return `env.ASSETS.fetch('/404.html')` with status 404 for unmatched non-API paths |
| `public/_headers` is applied to static assets | `curl -I https://staging.<domain>/pricing` shows the CSP and HSTS headers | Put the CSP in a `<meta http-equiv>` in `BaseLayout.astro`; HSTS comes from the Cloudflare zone setting |
| `build.format: 'file'` URLs are served without redirects | `/pricing`, `/app` and `/app/editor?resume=x` return 200 directly | Use `build.format: 'directory'` + `trailingSlash: 'always'` and update `SITE_PATHS`/`APP_ROUTES` with trailing slashes |
| Clerk React package name and `routing="hash"` | Check npm for `@clerk/react`; render `<SignIn routing="hash">` on staging | Use `@clerk/clerk-react` if that is the current package name. Keep hash routing. |
| Session token has an `email` claim | In the Clerk dashboard, Sessions → customize the token with `{"email":"{{user.primary_email_address}}"}`; decode a staging token | `getAuthProfile()` through the Backend API, in `meInit` and admin routes only |
| Clerk Hobby: email code + email link + Google enabled; email sending limits | Clerk dashboard on the prod instance | Google + email code only; a documented limit in `accounts-setup.md` |
| OpenAI snapshot ids exist | `GET /v1/models` with the staging key lists `gpt-5-nano-2025-08-07` and `gpt-5-mini-2025-08-07` | Use the aliases `gpt-5-nano` / `gpt-5-mini` in `MODELS` |
| `reasoning.effort: 'minimal'` is accepted | One staging call per model | Use `'low'` and raise `max_output_tokens` by 50% |
| D1 Free daily read and write row limits | The Cloudflare D1 pricing and limits pages | Keep the debounced autosave, and add a `costs` warning at 50% of the documented limit |
| Cron Triggers on Workers Free (2 needed) | `wrangler deploy` output for staging and prod | Run `runDaily` opportunistically from `/api/health` once per UTC day (guarded by an `app_settings` key `last_daily_run`, added as a skeleton addition) |
| Dodo event names, discount codes, mode detection, pre-tax amount field | Dodo docs + a test-mode purchase with a discount code | Grant on `payment.succeeded` only. The `fetchDodoPayment` re-fetch against the env's API base (`DODO_MODE`) is the authoritative mode check. |
| Razorpay `X-Razorpay-Event-Id` header is present | Test-mode webhook delivery | Dedupe on `(payment_id, event_type)` in `webhook_events.event_id` |
| Fonts embed `₹` and Indian names; text is selectable | `templates-golden.spec.ts` | Switch to another OFL family with ₹ coverage, and update `FONT_FAMILIES` and `public/fonts` only |
| GitHub Free private repo cannot use Environments | Repo Settings → Environments | Already decided: repository-level `_STAGING`/`_PROD` names work either way |
