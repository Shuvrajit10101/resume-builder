## 3. Data Model

*Status: spec part 03, written 2026-10-03. Appendix A (`parts/00-skeleton.md`) §2 holds the binding DDL, §6 the plan/limit/settings/event ids and §7 the resume JSON outline. This part adds only what Appendix A leaves open. Where this part and Appendix A disagree, Appendix A wins and the conflict is listed in "Skeleton issues" at the end.*

### 3.1 Data-layer rules

| Topic | Rule |
|---|---|
| Two databases | `DB_CORE` holds accounts, money, quotas and telemetry. `DB_DOCS` holds resume content. No SQL statement spans both databases. Cross-database work (export, erasure) runs one database at a time, docs first. |
| Access path | Only `src/server/data/**` touches a binding or Drizzle (Appendix A §8 #2). Handlers get `UserRepo` from `scoped(env, userId)`. Services get `SystemRepo` and `AdminRepo`. |
| Transactions | D1 has no interactive transactions, so `db.transaction()` is forbidden. ESLint `no-restricted-syntax` flags `.transaction(` inside `src/server/data/**`. When a write must be atomic across several statements, use `db.batch([...])`: a batch runs as one implicit transaction in one database. Prefer single-statement atomic patterns (guarded upsert, conditional `INSERT … SELECT … WHERE`, see 3.7 and 3.5.3) because they need one round trip. |
| Statement budget | Workers Free allows 50 D1 queries per invocation, and each statement inside a batch counts. Every endpoint must use **at most 25 statements** across both databases. The user-route prelude uses at most 6: user lookup, passes, settings and latest consents always; plan limits for the group and quota usage for the current period only when a service asks for them. `tests/helpers/db.ts` wraps the D1 bindings with a statement counter, and each integration test asserts `≤ 25` on the routes it calls. |
| Bound parameters | At most 100 per statement (D1 limit). Bulk statements are chunked. |
| CPU | Workers Free allows 10 ms CPU per request. List queries never select `data`, `jd_text`, `keywords` or `body`. JSON is parsed only for the one document being read or written. |
| JSON columns | Stored as `TEXT` from `JSON.stringify(value)` with no indentation. Every read goes through zod: `migrateResume` for resume `data`, `JdKeywordsSchema` for `keywords`, `EVENT_PROPS[name]` for `props`, `SettingsSchema.shape[key]` for `app_settings.value`. |
| Raw SQL | The Drizzle query builder is the default. The atomic statements in 3.5.3, 3.7, 3.8.3 and 3.10 use Drizzle's `sql` template (still a bound, prepared D1 statement), because their exact shape matters. String concatenation into SQL is forbidden (`check:boundaries` greps for `` sql.raw( `` and `` `${ `` inside `.prepare(`). |

### 3.2 Drizzle definition conventions

| Item | Convention |
|---|---|
| Files | `src/server/data/schema/core.ts` (15 tables) and `src/server/data/schema/docs.ts` (3 tables). |
| Table exports | camelCase plural of the SQL name: `users`, `consents`, `passes`, `checkoutIntents`, `webhookEvents`, `coupons`, `couponRedemptions`, `planLimits`, `appSettings`, `quotaUsage`, `aiBudgetDaily`, `llmCalls`, `ipCaps`, `events`, `accountDeletions`; `resumes`, `tailoredVersions`, `coverLetters`. |
| Row types | `export type <Singular>Row = typeof <table>.$inferSelect`: `UserRow`, `ConsentRow`, `PassRow`, `IntentRow`, `WebhookEventRow`, `CouponRow`, `CouponRedemptionRow`, `PlanLimitRow`, `AppSettingRow`, `QuotaUsageRow`, `AiBudgetDailyRow`, `LlmCallRow`, `IpCapRow`, `EventRow`, `AccountDeletionRow`, `ResumeRow`, `TailoredRow`, `CoverLetterRow`. These row types never leave `src/server`. API outputs are mapped explicitly into the `schemas/api.ts` shapes. |
| Columns | Always `text('snake_name')` or `integer('snake_name')` under a camelCase key (Appendix A §0). |
| Time | `integer('created_at')` as a plain number. Never use `{ mode: 'timestamp' }` or `Date`. |
| Booleans | `integer('active', { mode: 'boolean' })` plus `check('<table>_<col>_chk', sql\`${t.active} IN (0,1)\`)`. |
| Enums | `text('currency', { enum: ['USD','INR'] })` plus the CHECK constraint from Appendix A. Columns whose value set will grow (`plan_id`, `feature`, `limit_group`, `name`, `key`) get no CHECK. They are validated by zod in code (see 3.13 on frozen CHECK lists). |
| Defaults | `.default(0)` / `.default(1)` only where Appendix A has `DEFAULT`. Timestamps and ids never have SQL defaults: the repository always passes `now()` and `newId()`. |
| Uniques and indexes | Use `.unique()` on a single column and `unique('<name>').on(...)` for composite uniques. Indexes are declared as `index('<name>').on(...)`, using the names from Appendix A and 3.3. |

Pattern (every table follows it):

```ts
// src/server/data/schema/core.ts
import { sql } from 'drizzle-orm';
import { sqliteTable, text, integer, index, primaryKey, check } from 'drizzle-orm/sqlite-core';

export const passes = sqliteTable('passes', {
  id:                text('id').primaryKey(),
  userId:            text('user_id').notNull(),
  planId:            text('plan_id').notNull(),                                  // PlanId, zod-checked
  provider:          text('provider', { enum: ['dodo', 'razorpay', 'coupon'] }).notNull(),
  providerPaymentId: text('provider_payment_id').notNull().unique(),
  intentId:          text('intent_id'),
  couponId:          text('coupon_id'),
  amountMinor:       integer('amount_minor').notNull(),
  taxMinor:          integer('tax_minor').notNull().default(0),
  currency:          text('currency', { enum: ['USD', 'INR'] }).notNull(),
  purchasedAt:       integer('purchased_at').notNull(),
  revokedAt:         integer('revoked_at'),
  revokeReason:      text('revoke_reason', { enum: ['refund', 'dispute', 'fraud'] }),
  pseudonymisedAt:   integer('pseudonymised_at'),
}, (t) => [
  index('passes_user').on(t.userId),
  check('passes_provider_chk', sql`${t.provider} IN ('dodo','razorpay','coupon')`),
  check('passes_currency_chk', sql`${t.currency} IN ('USD','INR')`),
  check('passes_revoke_reason_chk', sql`${t.revokeReason} IS NULL OR ${t.revokeReason} IN ('refund','dispute','fraud')`),
  check('passes_amount_chk', sql`${t.amountMinor} >= 0 AND ${t.taxMinor} >= 0`),
  check('passes_revoked_pair_chk', sql`(${t.revokedAt} IS NULL) = (${t.revokeReason} IS NULL)`),
]);
export type PassRow = typeof passes.$inferSelect;
```

**Schema and migration parity.** `npm run db:generate` produced `migrations/*/0000_init.sql`. `check:migrations` also runs `drizzle-kit generate` for both configs into a temporary directory and fails if it produces any file. This means the TS schema and the committed SQL can never drift.

**Build-time check:** the generated `0000_init.sql` must contain every CHECK constraint from Appendix A and 3.3. If the pinned drizzle-kit drops `check()` output, hand-edit `0000_init.sql` to add the CHECKs. This is allowed only before the file is first applied to any remote database. Keep the `check()` declarations in TS anyway.

### 3.3 Constraints and indexes beyond Appendix A

Appendix A's DDL plus the following goes into the two `0000_init.sql` files. None of these adds a column or a table.

**Extra CHECK constraints**

| Table | Constraint |
|---|---|
| `users` | `country IS NULL OR length(country) = 2`; `length(last_seen_day) = 10`; `campus_code IS NULL OR (length(campus_code) BETWEEN 2 AND 20 AND campus_code = upper(campus_code))` |
| `passes` | `amount_minor >= 0 AND tax_minor >= 0`; `(revoked_at IS NULL) = (revoke_reason IS NULL)` |
| `checkout_intents` | `amount_minor >= 0 AND amount_minor <= list_amount_minor` |
| `coupons` | `length(code) BETWEEN 4 AND 20 AND code NOT GLOB '*[^A-Z0-9]*'`; `max_redemptions >= 1`; `redemptions >= 0` (it may exceed `max_redemptions` after a paid-coupon race, see 3.10) |
| `quota_usage` | `used >= 0` |
| `ai_budget_daily` | every `*_micro_usd` and `*calls` column `>= 0` |
| `ip_caps` | `count >= 1`; bucket list widened to `('checker_ai','events_public','coupon_attempts')` (skeleton addition) |
| `llm_calls` | every token column and `cost_micro_usd`, `latency_ms` `>= 0` |
| `resumes` | `length(title) BETWEEN 1 AND 80`; `revision >= 1`; `size_bytes BETWEEN 2 AND 65536` |
| `tailored_versions` | `length(title) BETWEEN 1 AND 80`; `length(job_title) <= 120`; `length(company) <= 120`; `length(jd_text) BETWEEN 1 AND 15000`; `match_percent IS NULL OR match_percent BETWEEN 0 AND 100`; `revision >= 1`; `size_bytes BETWEEN 2 AND 65536` |
| `cover_letters` | `length(title) BETWEEN 1 AND 80`; `length(body) <= 6000`; `revision >= 1` |

SQLite `length()` counts code points, while zod `.max()` counts UTF-16 units, which is never fewer. A value that passes zod therefore always passes the SQL CHECK.

**Extra indexes**

| Index | Why |
|---|---|
| `checkout_intents_status_created ON checkout_intents (status, created_at)` | The daily job expires `created` intents older than 48 h without a full scan |
| `webhook_events_received ON webhook_events (received_at)` | 90-day purge |
| `events_created ON events (created_at)` | 400-day purge (D1 bills rows read, so purges must not scan) |
| `coupon_redemptions_user ON coupon_redemptions (user_id)` | Export, erasure and the `already_used` check |

Existing unique constraints that the code relies on: `users.clerk_user_id` (ensure), `passes.provider_payment_id` (grant idempotency), `checkout_intents.provider_ref` (Razorpay order lookup), `coupons.code`, `coupon_redemptions (coupon_id, user_id)`, and the composite primary keys of `quota_usage`, `plan_limits`, `ip_caps` and `webhook_events` (atomic upserts and dedupe).

### 3.4 Resume JSON (`src/shared/schemas/resume.ts`)

#### 3.4.1 Field limits

All limits live in `RESUME_LIMITS` in `src/shared/config/input-limits.ts` (skeleton addition), next to the existing `MAX_*` constants. Lengths count UTF-16 units (JS `string.length`). "line" means a single-line string: no characters U+0000–U+001F or U+007F. "multiline" allows `\n` only. The browser runs `stripControlChars` and `normalizeWhitespace` on input, so users never see these errors.

| Path | Type | Limit |
|---|---|---|
| `settings.sectionOrder` | `SectionKey[]` | a permutation of all 10 built-in keys plus `custom:<id>` for each custom section (10–13 entries) |
| `settings.hiddenSections` | `SectionKey[]` | subset of `sectionOrder`, no duplicates |
| `settings.sectionTitles` | partial record, built-in keys only | line ≤ 40 each |
| `settings.fontScale` | number | one of `0.9, 0.95, 1, 1.05, 1.1` |
| `basics.fullName` | line | ≤ 80 |
| `basics.headline` | line | ≤ 120 |
| `basics.email` | `''` or email | ≤ 254 |
| `basics.phone` | line, `^[0-9+().\- ]*$` | ≤ 30 |
| `basics.location.city` / `region` / `country` | line | ≤ 60 each |
| `basics.links` | array | ≤ 5; `label` line ≤ 40; `url` `''` or `https://…` ≤ 300 |
| `summary.text` | multiline | ≤ 1,200 (`MAX_SUMMARY_CHARS`) |
| Any `Bullet.text` | line | ≤ 400 (`MAX_BULLET_CHARS`); an empty string is allowed (the renderer skips it) |
| `experience`, `internships`, `positions` | array | ≤ 30 items each; `organization` ≤ 100, `title` ≤ 100, `location` ≤ 80; `bullets` ≤ 12 |
| `education` | array | ≤ 10; `institution` ≤ 120, `degree` ≤ 100, `field` ≤ 100, `board` ≤ 60, `location` ≤ 80; `score.value` 1–10 chars, `score.outOf` 0–10 chars; `highlights` ≤ 6; `coursework` ≤ 12 items of 1–60 |
| `projects` | array | ≤ 20; `name` ≤ 100, `role` ≤ 80, `url` `''` or https ≤ 300; `techStack` ≤ 20 items of 1–40; `bullets` ≤ 8 |
| `skills` | array | ≤ 12 groups; `label` ≤ 40; `items` ≤ 40 of 1–50 |
| `certifications` | array | ≤ 30; `name` ≤ 120, `issuer` ≤ 100, `credentialId` ≤ 80, `url` `''` or https ≤ 300 |
| `achievements` | array | ≤ 30; `title` ≤ 150; `description` multiline ≤ 300 |
| `languages` | array | ≤ 10; `name` ≤ 40 |
| `custom` | array | ≤ 3; `title` line 1–40; `items` ≤ 20; `heading` ≤ 120, `subheading` ≤ 120; `bullets` ≤ 8 |
| Item ids (`Id`) | `ItemIdSchema` | `^[A-Za-z0-9_-]{1,40}$` (`crypto.randomUUID()` fits). Unique across the whole document. |
| `YearMonth` | `YearMonthSchema` | `^(19|20)\d{2}(-(0[1-9]|1[0-2]))?$` |
| Whole document | `resumeByteSize(data)` | ≤ 65,536 UTF-8 bytes of `JSON.stringify(data)` (`MAX_RESUME_BYTES`) |

#### 3.4.2 zod outline (zod 4, exact pinned version)

The outline shows the limits as literal numbers so they are easy to read. The implementation must read every one of them from `RESUME_LIMITS` (3.4.1), and `resume-schema.test.ts` checks the limits at their boundaries (N passes, N+1 fails) for every row of the table.

```ts
import { z } from 'zod';
import { TEMPLATE_IDS } from '@shared/config/templates';
import { MAX_RESUME_BYTES, MAX_BULLET_CHARS, MAX_SUMMARY_CHARS, RESUME_LIMITS as L } from '@shared/config/input-limits';
import { ItemIdSchema, YearMonthSchema } from './common';

export const RESUME_SCHEMA_VERSION = 1;
export const BUILT_IN_SECTION_KEYS = ['summary','experience','internships','positions','education',
  'projects','skills','certifications','achievements','languages'] as const;
const FONT_SCALES = [0.9, 0.95, 1, 1.05, 1.1] as const;

const SINGLE = /[\u0000-\u001F\u007F]/;
const MULTI  = /[\u0000-\u0009\u000B-\u001F\u007F]/;
const line      = (max: number) => z.string().max(max).refine((s) => !SINGLE.test(s), 'control_chars');
const multiline = (max: number) => z.string().max(max).refine((s) => !MULTI.test(s), 'control_chars');
const httpsOrEmpty = z.string().max(L.urlMax).regex(/^(|https:\/\/[^\s<>"'`]+)$/);
const emailOrEmpty = z.union([z.literal(''), z.email().max(254)]);

export const SectionKeySchema = z.union([
  z.enum(BUILT_IN_SECTION_KEYS),
  z.string().regex(/^custom:[A-Za-z0-9_-]{1,40}$/) as unknown as z.ZodType<`custom:${string}`>,
]);
export const DateRangeSchema = z.strictObject({
  start: YearMonthSchema.nullable(), end: YearMonthSchema.nullable(), current: z.boolean(),
}).refine((d) => !d.current || d.end === null, { path: ['end'], message: 'end_must_be_null_when_current' });
export const BulletSchema = z.strictObject({ id: ItemIdSchema, text: line(MAX_BULLET_CHARS) });

const ExperienceItemSchema = z.strictObject({
  id: ItemIdSchema, organization: line(100), title: line(100), location: line(80),
  employmentType: z.enum(['full_time','part_time','contract','internship','freelance','volunteer','other']).nullable(),
  dates: DateRangeSchema, bullets: z.array(BulletSchema).max(12),
});
const EducationItemSchema = z.strictObject({
  id: ItemIdSchema, institution: line(120),
  level: z.enum(['secondary','senior_secondary','diploma','bachelor','master','doctorate','other']),
  degree: line(100), field: line(100), board: line(60), location: line(80), dates: DateRangeSchema,
  score: z.strictObject({ kind: z.enum(['cgpa','gpa','percentage','grade']),
                          value: line(10).min(1), outOf: line(10) }).nullable(),
  highlights: z.array(BulletSchema).max(6),
  coursework: z.array(line(60).min(1)).max(12),
});
const ProjectItemSchema = z.strictObject({
  id: ItemIdSchema, name: line(100), role: line(80), url: httpsOrEmpty, dates: DateRangeSchema,
  techStack: z.array(line(40).min(1)).max(20), bullets: z.array(BulletSchema).max(8),
});
const SkillGroupSchema = z.strictObject({ id: ItemIdSchema, label: line(40), items: z.array(line(50).min(1)).max(40) });
const CertificationItemSchema = z.strictObject({ id: ItemIdSchema, name: line(120), issuer: line(100),
  date: YearMonthSchema.nullable(), credentialId: line(80), url: httpsOrEmpty });
const AchievementItemSchema = z.strictObject({ id: ItemIdSchema, title: line(150),
  date: YearMonthSchema.nullable(), description: multiline(300) });
const LanguageItemSchema = z.strictObject({ id: ItemIdSchema, name: line(40),
  proficiency: z.enum(['native','fluent','professional','intermediate','basic']).nullable() });
const CustomSectionSchema = z.strictObject({ id: ItemIdSchema, title: line(40).min(1),
  items: z.array(z.strictObject({ id: ItemIdSchema, heading: line(120), subheading: line(120),
    dates: DateRangeSchema.nullable(), bullets: z.array(BulletSchema).max(8) })).max(20) });

export const ResumeDataSchema = z.strictObject({
  schemaVersion: z.literal(RESUME_SCHEMA_VERSION),
  settings: z.strictObject({
    templateId: z.enum(TEMPLATE_IDS),
    pageSize: z.enum(['A4','LETTER']),
    locale: z.enum(['en-IN','en-US','en-GB']),
    dateFormat: z.enum(['MMM YYYY','MM/YYYY','YYYY']),
    fontScale: z.number().refine((v) => (FONT_SCALES as readonly number[]).includes(v), 'font_scale'),
    margins: z.enum(['normal','narrow']),
    sectionOrder: z.array(SectionKeySchema).max(13),
    hiddenSections: z.array(SectionKeySchema).max(13),
    sectionTitles: z.partialRecord(z.enum(BUILT_IN_SECTION_KEYS), line(40)),
  }),
  basics: z.strictObject({
    fullName: line(80), headline: line(120), email: emailOrEmpty,
    phone: line(30).regex(/^[0-9+().\- ]*$/),
    location: z.strictObject({ city: line(60), region: line(60), country: line(60) }),
    links: z.array(z.strictObject({ id: ItemIdSchema, label: line(40), url: httpsOrEmpty })).max(5),
  }),
  summary: z.strictObject({ text: multiline(MAX_SUMMARY_CHARS) }),
  experience: z.array(ExperienceItemSchema).max(30),
  internships: z.array(ExperienceItemSchema).max(30),
  positions: z.array(ExperienceItemSchema).max(30),
  education: z.array(EducationItemSchema).max(10),
  projects: z.array(ProjectItemSchema).max(20),
  skills: z.array(SkillGroupSchema).max(12),
  certifications: z.array(CertificationItemSchema).max(30),
  achievements: z.array(AchievementItemSchema).max(30),
  languages: z.array(LanguageItemSchema).max(10),
  custom: z.array(CustomSectionSchema).max(3),
}).superRefine(checkDocumentInvariants);
export type ResumeData = z.infer<typeof ResumeDataSchema>;

export function resumeByteSize(data: ResumeData): number {
  return new TextEncoder().encode(JSON.stringify(data)).length;
}
```

`checkDocumentInvariants(data, ctx)` adds one issue for each violation:

1. `sectionOrder` contains each built-in key exactly once, plus `custom:<id>` for each `custom[i].id` exactly once, and nothing else (path `['settings','sectionOrder']`).
2. `hiddenSections` has no duplicates and is a subset of `sectionOrder`.
3. Every `id` in the document is unique: links, all items, bullets, highlights, custom sections and custom items. The issue path points at the second occurrence.

The byte size is **not** part of the zod schema. `services/documents.ts` calls `resumeByteSize` after the zod parse and throws `PAYLOAD_TOO_LARGE` with `details: { maxBytes: 65536, sizeBytes }`, so the API returns 413 instead of 400.

**Rejected fields.** Every object is strict, so unknown keys fail. `resume-schema.test.ts` asserts that each of `photo`, `photoUrl`, `dateOfBirth`, `dob`, `age`, `gender`, `maritalStatus`, `religion`, `nationality`, `fatherName`, placed at the root, in `basics` or in an item, gives `VALIDATION_FAILED` / a zod error.

**Rendering rules the schema implies.** These are shared by the PDF, the DOCX and `resumeToPlainText`. A section renders when it is not in `hiddenSections` **and** has content. Empty bullets, links with an empty `url` and items whose text fields are all empty are skipped. `basics` always renders first and is not a section.

**`emptyResume(templateId)`**:
- `sectionOrder` is the template's default order (Appendix A §7.2), then the remaining built-in keys in `BUILT_IN_SECTION_KEYS` order.
- `hiddenSections: []` and `sectionTitles: {}`.
- `locale`: `en-IN` for the `india` family, `en-US` for `global`. The browser overrides it from `navigator.language` when that is `en-GB` or `en-IN`.
- `pageSize`: A4 unless the locale is `en-US`.
- `dateFormat: 'MMM YYYY'`, `fontScale: 1`, `margins: 'normal'`.
- Every string is `''`, every array is `[]`, and `summary.text` is `''`.

**`migrateResume(raw: unknown): ResumeData`** is the only reader for stored data. It accepts a string (it `JSON.parse`s it) or an object. While `schemaVersion < RESUME_SCHEMA_VERSION` it applies `MIGRATIONS[v]` (an internal map from `n` to `n+1`), then runs `ResumeDataSchema.parse`. A version higher than the code knows, or a parse failure, throws. The server maps that throw to `INTERNAL` and logs `resume_parse_failed` with only `{docKind, docId}`. Migration policy:
- Any shape change bumps `RESUME_SCHEMA_VERSION` and adds `MIGRATIONS[n]` plus a fixture `tests/fixtures/resumes/v<n>-*.json` that must migrate cleanly.
- Rows are migrated lazily on read. The next save writes the new version and `schema_version`. There is no bulk rewrite, so there are no write spikes.
- API request bodies are always parsed with the **current** `ResumeDataSchema`. The browser migrates IndexedDB drafts through `migrateResume` before it sends them.

#### 3.4.3 Complete example: Indian fresher (valid against 3.4.2)

The person and the organisations below are fictional. `src/shared/samples/sample-resume.ts` (`SAMPLE_RESUME_INDIA`) uses this document, and so does `tests/fixtures/resumes/india-fresher.json`. Short readable ids are valid because `ItemIdSchema` allows them; the editor generates UUIDs.

```json
{
  "schemaVersion": 1,
  "settings": {
    "templateId": "india_fresher",
    "pageSize": "A4",
    "locale": "en-IN",
    "dateFormat": "MMM YYYY",
    "fontScale": 1,
    "margins": "normal",
    "sectionOrder": ["summary", "education", "skills", "projects", "internships", "positions",
                     "achievements", "certifications", "experience", "languages"],
    "hiddenSections": ["experience"],
    "sectionTitles": { "summary": "Career Objective", "positions": "Positions of Responsibility" }
  },
  "basics": {
    "fullName": "Priya Raghavan",
    "headline": "B.Tech CSE 2026 | Backend and data",
    "email": "priya.raghavan@example.com",
    "phone": "+91 90000 00000",
    "location": { "city": "Pune", "region": "Maharashtra", "country": "India" },
    "links": [
      { "id": "lnk-1", "label": "GitHub", "url": "https://github.com/priya-raghavan-example" },
      { "id": "lnk-2", "label": "LinkedIn", "url": "https://www.linkedin.com/in/priya-raghavan-example" }
    ]
  },
  "summary": {
    "text": "Final-year Computer Science student with one backend internship and two shipped projects in Node.js and Python. Looking for an entry-level backend or data engineering role."
  },
  "experience": [],
  "internships": [
    {
      "id": "int-1",
      "organization": "Tarangini Software Labs Pvt. Ltd.",
      "title": "Software Engineering Intern",
      "location": "Pune (hybrid)",
      "employmentType": "internship",
      "dates": { "start": "2025-05", "end": "2025-07", "current": false },
      "bullets": [
        { "id": "b-1", "text": "Built 9 REST endpoints in Node.js and PostgreSQL for an internal invoice tracker used by 40 finance staff." },
        { "id": "b-2", "text": "Cut monthly report generation from 12 minutes to 90 seconds by adding indexes and caching query results." },
        { "id": "b-3", "text": "Wrote 65 Jest unit tests, raising billing-module coverage from 48% to 81%." }
      ]
    }
  ],
  "positions": [
    {
      "id": "pos-1",
      "organization": "Coding Club, Deccan Valley Institute of Technology",
      "title": "Technical Lead",
      "location": "Pune",
      "employmentType": "volunteer",
      "dates": { "start": "2024-08", "end": null, "current": true },
      "bullets": [
        { "id": "b-4", "text": "Organised a 24-hour hackathon with 180 participants and 12 volunteer mentors." },
        { "id": "b-5", "text": "Run weekly data-structures practice sessions for 60 second-year students." }
      ]
    }
  ],
  "education": [
    {
      "id": "edu-1",
      "institution": "Deccan Valley Institute of Technology",
      "level": "bachelor",
      "degree": "B.Tech",
      "field": "Computer Science and Engineering",
      "board": "",
      "location": "Pune",
      "dates": { "start": "2022-08", "end": "2026-05", "current": false },
      "score": { "kind": "cgpa", "value": "8.42", "outOf": "10" },
      "highlights": [ { "id": "b-6", "text": "Department rank 7 of 120 in semester 5." } ],
      "coursework": ["Data Structures", "Database Management Systems", "Operating Systems", "Computer Networks"]
    },
    {
      "id": "edu-2",
      "institution": "Godavari Valley Junior College",
      "level": "senior_secondary",
      "degree": "Class XII",
      "field": "Science (PCM)",
      "board": "CBSE",
      "location": "Nashik",
      "dates": { "start": null, "end": "2022", "current": false },
      "score": { "kind": "percentage", "value": "91.2", "outOf": "100" },
      "highlights": [],
      "coursework": []
    },
    {
      "id": "edu-3",
      "institution": "Godavari Valley Public School",
      "level": "secondary",
      "degree": "Class X",
      "field": "",
      "board": "CBSE",
      "location": "Nashik",
      "dates": { "start": null, "end": "2020", "current": false },
      "score": { "kind": "percentage", "value": "94.6", "outOf": "100" },
      "highlights": [],
      "coursework": []
    }
  ],
  "projects": [
    {
      "id": "prj-1",
      "name": "Placement Tracker",
      "role": "Team lead (3 members)",
      "url": "https://github.com/priya-raghavan-example/placement-tracker",
      "dates": { "start": "2025-01", "end": "2025-04", "current": false },
      "techStack": ["React", "Node.js", "PostgreSQL", "Docker"],
      "bullets": [
        { "id": "b-7", "text": "Built a web app that tracks 300+ campus job applications with deadline reminders." },
        { "id": "b-8", "text": "Designed the PostgreSQL schema and 20 validated REST endpoints." }
      ]
    },
    {
      "id": "prj-2",
      "name": "Crop Price Forecaster",
      "role": "Solo project",
      "url": "",
      "dates": { "start": "2024-09", "end": "2024-11", "current": false },
      "techStack": ["Python", "pandas", "scikit-learn"],
      "bullets": [
        { "id": "b-9", "text": "Trained a gradient-boosting model on 5 years of mandi price data, reaching 8.3% mean absolute percentage error." }
      ]
    }
  ],
  "skills": [
    { "id": "sk-1", "label": "Languages", "items": ["Java", "Python", "JavaScript", "SQL"] },
    { "id": "sk-2", "label": "Frameworks and tools", "items": ["React", "Node.js", "Express", "Git", "Docker", "Linux"] },
    { "id": "sk-3", "label": "Databases", "items": ["PostgreSQL", "MySQL", "MongoDB"] }
  ],
  "certifications": [
    { "id": "cert-1", "name": "Programming in Java", "issuer": "NPTEL", "date": "2024-04", "credentialId": "", "url": "" }
  ],
  "achievements": [
    { "id": "ach-1", "title": "Winner, DVIT CodeSprint 2024", "date": "2024-12",
      "description": "Team of 4; built an offline-first complaint-logging app for a college transport problem statement." }
  ],
  "languages": [
    { "id": "lang-1", "name": "English", "proficiency": "fluent" },
    { "id": "lang-2", "name": "Hindi", "proficiency": "fluent" },
    { "id": "lang-3", "name": "Marathi", "proficiency": "native" }
  ],
  "custom": []
}
```

#### 3.4.4 AI-related shapes as zod

Appendix A §7.3 gives the types. The schemas also live in `schemas/resume.ts` as `FieldRefSchema` (already listed), `KeywordSchema`, `JdKeywordsSchema`, `MatchResultSchema` and `HonestyFlagSchema` (skeleton additions; the API outputs reuse them). Limits: `Keyword.term` line 1–60; `aliases` ≤ 5 of 1–60; `roleTitle` ≤ 120; `required` and `preferred` ≤ 25 each. `HonestyFlag.text` ≤ 200, with `0 ≤ start < end ≤ length of the flagged text`.

### 3.5 Master resumes, tailored versions and cover letters

#### 3.5.1 Lifecycle

| Operation | Data rule |
|---|---|
| Create master | `source` = `blank` (`emptyResume`), `import` (client-confirmed draft), `sample` (`SAMPLE_RESUME_INDIA` for the `india` template family, otherwise `SAMPLE_RESUME_GLOBAL`, with `settings.templateId` set to the requested template). `revision = 1`. |
| Duplicate master | Copies the stored `data` text and `schema_version` as they are (no parse), with `source = 'duplicate'`, `revision = 1` and title `capText(title + ' (copy)', 80)`. Tailored versions are **not** copied. |
| Create tailored | A one-time snapshot copy of the master's stored `data` and `schema_version`. There is no live link: later master edits never reach the tailored copy and the reverse. `title` defaults to `capText([jobTitle, company].filter(Boolean).join(' - ') || 'Tailored version', 80)`. `keywords*` and `match_percent` start NULL. |
| JD hash | `jd_hash = sha256Hex(normalizeWhitespace(jdText.normalize('NFKC')).toLowerCase())`, computed on the server whenever `jd_text` is written. |
| Keyword cache | Valid only when `keywords_jd_hash = jd_hash`. `aiJdKeywords` writes `keywords`, `keywords_jd_hash = jd_hash` and `keywords_model`, and recomputes `match_percent` from the stored `data` in the same UPDATE. A JD edit (`tailoredUpdateMeta`) changes `jd_hash`, sets `match_percent = NULL` and leaves the stale `keywords` in place (the API reports `keywordsStale: true`). |
| Match recompute | `tailoredSave` sets `match_percent = computeMatch(keywords, data).percent` when the cache is valid, otherwise NULL. The server never trusts a client-sent percent. |
| Cover letter | Belongs to one tailored version. `body` is normalized on write: `\r\n` → `\n`, three or more newlines → `\n\n`, trimmed, ≤ 6,000. `revision` covers `title` and `body`. |
| Delete | Run in `services/documents.ts`, children first: `coverLetters.deleteByTailored` → `tailored.deleteByResume` / `tailored.delete` → `resumes.delete`. Each statement is idempotent and filters on `user_id`. If one step fails, the caller retries the whole request, and the children-first order means no orphan is ever visible. |

#### 3.5.2 Revisions (optimistic concurrency)

`revision` covers only the content columns: `data` for masters and tailored versions, `title` + `body` for cover letters. Renames and metadata edits (`title`, `job_title`, `company`, `jd_text`) do not bump it but do update `updated_at`.

```sql
-- resumes.save (scoped): one statement, then inspect meta.changes
UPDATE resumes SET data = ?3, schema_version = ?4, size_bytes = ?5, revision = revision + 1, updated_at = ?6
 WHERE id = ?1 AND user_id = ?2 AND revision = ?7
RETURNING revision, updated_at;
-- no row returned → SELECT revision FROM resumes WHERE id = ?1 AND user_id = ?2
--   no row → NOT_FOUND;  row → REVISION_CONFLICT { currentRevision }
```

`tailored_versions.save` has the same shape and also sets `match_percent`. `cover_letters.save` has the same shape over `title`/`body`.

#### 3.5.3 Count caps without races

Creation is a single conditional insert, so two parallel requests cannot both pass the cap:

```sql
INSERT INTO resumes (id, user_id, title, source, data, schema_version, revision, size_bytes, created_at, updated_at)
SELECT ?1, ?2, ?3, ?4, ?5, ?6, 1, ?7, ?8, ?8
 WHERE (SELECT count(*) FROM resumes WHERE user_id = ?2) < ?9;      -- ?9 = resumes_max
-- meta.changes = 0 → PAYWALL { reason: 'count_cap', cap: 'resumes_max', limit }
```

`tailored.create` uses `INSERT … SELECT … FROM resumes r WHERE r.id = ?resumeId AND r.user_id = ?userId AND (SELECT count(*) FROM tailored_versions WHERE user_id = ?userId) < ?cap`. It copies `r.data`, `r.schema_version` and `r.size_bytes` inside the database, so 64 KB never travels through the Worker. If `changes = 0`, one follow-up read decides between `NOT_FOUND` (no master) and `PAYWALL`. `coverLetters.create` works the same way, checking that the tailored version is owned and counting against `cover_letters_max`.

### 3.6 Passes ledger

| Rule | Detail |
|---|---|
| Writers | INSERT only by `grantPass` in `billing/grant.ts` (through `data/ledger.ts`). UPDATE `revoked_at`/`revoke_reason` only by `revokePass` (`… WHERE id = ? AND revoked_at IS NULL`, once). UPDATE `user_id`/`pseudonymised_at` only by `eraseUserData` (3.15). No DELETE, ever. |
| Idempotency | `INSERT INTO passes (…) VALUES (…) ON CONFLICT (provider_payment_id) DO NOTHING RETURNING id`. If no row comes back, the payment was already granted: read the existing row and return `{ passId, created: false }`. Side effects (`payment_succeeded` / `coupon_redeemed` event, intent → `paid`, coupon counters) run only when `created: true`, in the same `db.batch`. |
| `provider_payment_id` | Dodo `payment_id` · Razorpay `pay_…` · `coupon:<coupon_redemptions.id>` |
| `amount_minor` | Dodo: the pre-tax amount from the re-fetched payment. Razorpay: the captured amount in paise (GST-inclusive list price minus our coupon). Coupon: `0`. |
| `tax_minor` | Dodo: the tax from the re-fetched payment. Razorpay and coupon: `0`. |
| `purchased_at` | Provider payment time: Dodo `created_at` (ISO string parsed to epoch ms), Razorpay `created_at` (unix seconds × 1000). If it is missing, `now()`. |
| Revocation | A full refund → `refund`. A dispute opened → `dispute`. `fraud` is only set by hand (D1 console, recorded in the incident note). A partial refund does not revoke. A won dispute does not re-grant automatically; the founder issues a 100%-off coupon for the same plan instead (the only grant path besides payment). |
| Access | Derived only by `computeAccess(passes where revoked_at IS NULL …)`. A revoked segment disappears from the stack, and later segments move earlier. |
| Lifetime count | `SELECT count(*) FROM passes WHERE plan_id IN ('inr_lifetime','usd_lifetime') AND revoked_at IS NULL`. It counts coupon-granted lifetime passes too. |
| Retention | Kept for at least 8 years (Indian accounting records). After account deletion the row is pseudonymised, not deleted. |

**`checkout_intents.status` transitions**:
- `created` → `paid` (grant), `failed` (provider failure event), or `expired` (daily job, older than 48 h).
- `failed` → `paid` or `expired`.
- `expired` → `paid`: a late payment is still honoured.
- `paid` is terminal.

Every transition is a guarded UPDATE: `… SET status = ?, updated_at = ? WHERE id = ? AND status IN (<allowed from-states>)`. When a payment matches an intent whose `plan_id`, `currency` or amount differ from the re-fetched payment, the grant is refused: `webhook_events.outcome = 'rejected'` plus `log.error('payment_mismatch')`, which alerts through Sentry.

### 3.7 Quota counters and the AI budget

**Per-user period quota** (`services/quota.ts` → `repo.quotas`):

```sql
-- reserve(feature, periodKey, limit)   (limit ≥ 1; limit 0 → PAYWALL before any SQL)
INSERT INTO quota_usage (user_id, feature, period_key, used, updated_at) VALUES (?1, ?2, ?3, 1, ?4)
ON CONFLICT (user_id, feature, period_key) DO UPDATE SET used = used + 1, updated_at = excluded.updated_at
 WHERE quota_usage.used < ?5
RETURNING used;                     -- no row → QUOTA_EXHAUSTED { feature, limit, periodEndsAt }

-- refund(feature, periodKey)
UPDATE quota_usage SET used = used - 1, updated_at = ?4
 WHERE user_id = ?1 AND feature = ?2 AND period_key = ?3 AND used > 0;
```

- `Reservation = { feature, periodKey, usedAfter, limit, periodEndsAt }`. A refund happens only on `AI_PROVIDER_ERROR`, `AI_OUTPUT_INVALID` or an `AI_BUDGET_EXHAUSTED` that occurs after the quota was reserved.
- Rows from past periods are kept until the account is deleted (at most 5 rows per user per period). The snapshot reads only the current `periodKey`.
- A limit lowered below `used` simply blocks further use. Raising a limit takes effect on the next request (settings and limits are read once per request and never cached across requests).

**Global daily budget** (`systemRepo.budget`, day = `utcDay(now())` at reservation time and carried in the reservation):

```sql
-- reserve(day, est, budget, anon, anonBudget); est ≤ budget (and est ≤ anonBudget for anon) is checked in code first
INSERT INTO ai_budget_daily (day, reserved_micro_usd, anon_reserved_micro_usd, updated_at)
VALUES (?1, ?2, CASE WHEN ?4 = 1 THEN ?2 ELSE 0 END, ?3)
ON CONFLICT (day) DO UPDATE SET
  reserved_micro_usd = reserved_micro_usd + ?2,
  anon_reserved_micro_usd = anon_reserved_micro_usd + CASE WHEN ?4 = 1 THEN ?2 ELSE 0 END,
  updated_at = ?3
 WHERE spent_micro_usd + reserved_micro_usd + ?2 <= ?5
   AND (?4 = 0 OR anon_spent_micro_usd + anon_reserved_micro_usd + ?2 <= ?6)
RETURNING day;                      -- no row → AI_BUDGET_EXHAUSTED

-- settle(day, est, actual, anon): always runs, success or failure
UPDATE ai_budget_daily SET
  reserved_micro_usd = max(0, reserved_micro_usd - ?2),
  spent_micro_usd = spent_micro_usd + ?3,
  anon_reserved_micro_usd = max(0, anon_reserved_micro_usd - CASE WHEN ?4 = 1 THEN ?2 ELSE 0 END),
  anon_spent_micro_usd = anon_spent_micro_usd + CASE WHEN ?4 = 1 THEN ?3 ELSE 0 END,
  calls = calls + 1, anon_calls = anon_calls + ?4, updated_at = ?5
 WHERE day = ?1;
```

`actual` is `costMicroUsd(model, usage)`. When the provider gives no usage (timeout or network error), `actual` is the reserved worst-case estimate. This is conservative because OpenAI may still bill the call. A Worker crash between reserve and settle leaves `reserved_micro_usd` inflated for that day only, so it fails safe.

### 3.8 `plan_limits` and `app_settings`

#### 3.8.1 Seed migration `migrations/core/0001_seed_defaults.sql` (custom drizzle migration)

```sql
-- updated_at = 1790985600000 (2026-10-03T00:00:00Z)
INSERT OR IGNORE INTO plan_limits (limit_group, feature, max_value, updated_at) VALUES
 ('free','ai_action',5,1790985600000),      ('pass_7d','ai_action',60,1790985600000),
 ('pass_30d','ai_action',200,1790985600000), ('pass_90d','ai_action',600,1790985600000),
 ('lifetime','ai_action',150,1790985600000),
 ('free','ai_import',1,1790985600000),      ('pass_7d','ai_import',5,1790985600000),
 ('pass_30d','ai_import',10,1790985600000),  ('pass_90d','ai_import',20,1790985600000),
 ('lifetime','ai_import',10,1790985600000),
 ('free','jd_analysis',1,1790985600000),    ('pass_7d','jd_analysis',20,1790985600000),
 ('pass_30d','jd_analysis',60,1790985600000),('pass_90d','jd_analysis',150,1790985600000),
 ('lifetime','jd_analysis',50,1790985600000),
 ('free','pdf_export',1,1790985600000),     ('pass_7d','pdf_export',100,1790985600000),
 ('pass_30d','pdf_export',300,1790985600000),('pass_90d','pdf_export',900,1790985600000),
 ('lifetime','pdf_export',300,1790985600000),
 ('free','docx_export',0,1790985600000),    ('pass_7d','docx_export',100,1790985600000),
 ('pass_30d','docx_export',300,1790985600000),('pass_90d','docx_export',900,1790985600000),
 ('lifetime','docx_export',300,1790985600000),
 ('free','resumes_max',2,1790985600000),    ('pass_7d','resumes_max',10,1790985600000),
 ('pass_30d','resumes_max',10,1790985600000),('pass_90d','resumes_max',20,1790985600000),
 ('lifetime','resumes_max',20,1790985600000),
 ('free','tailored_max',1,1790985600000),   ('pass_7d','tailored_max',20,1790985600000),
 ('pass_30d','tailored_max',50,1790985600000),('pass_90d','tailored_max',100,1790985600000),
 ('lifetime','tailored_max',100,1790985600000),
 ('free','cover_letters_max',0,1790985600000),('pass_7d','cover_letters_max',20,1790985600000),
 ('pass_30d','cover_letters_max',50,1790985600000),('pass_90d','cover_letters_max',100,1790985600000),
 ('lifetime','cover_letters_max',100,1790985600000);

INSERT OR IGNORE INTO app_settings (key, value, updated_at, updated_by) VALUES
 ('ai_enabled','true',1790985600000,'migration'),
 ('ai_daily_budget_micro_usd','1500000',1790985600000,'migration'),
 ('anon_ai_enabled','true',1790985600000,'migration'),
 ('anon_ai_daily_budget_micro_usd','250000',1790985600000,'migration'),
 ('anon_checker_ip_daily_limit','3',1790985600000,'migration'),
 ('events_ip_daily_limit','300',1790985600000,'migration'),
 ('coupon_ip_daily_limit','20',1790985600000,'migration'),
 ('checkout_usd_enabled','false',1790985600000,'migration'),
 ('checkout_inr_enabled','false',1790985600000,'migration'),
 ('lifetime_founder_limit','200',1790985600000,'migration'),
 ('dodo_max_discount_percent','50',1790985600000,'migration'),
 ('read_only','false',1790985600000,'migration'),
 ('banner_message','""',1790985600000,'migration');
```

`tests/unit/shared/plans.test.ts` reads this file and asserts that it equals `DEFAULT_PLAN_LIMITS` (40 rows) and `DEFAULT_SETTINGS` (13 keys) exactly.

#### 3.8.2 Reading, invalid values and changes

- `getSettings(sys)` reads all `app_settings` rows (one statement) and parses each key with `SettingsSchema.shape[key]` from `JSON.parse(value)`.
  - **Missing or invalid value** → the type's zero value: booleans `false`, integers `0`, strings `""`. It also calls `log.error('setting_invalid', { key })`, which goes to Sentry. Zero values fail closed: AI off, budgets 0, checkout off, no lifetime sales, no Dodo discount accepted, IP limits 0. `read_only` falls back to `false`.
  - Unknown keys are ignored.
- `getLimits(sys, group)` reads the group's rows (one statement). A missing row → `DEFAULT_PLAN_LIMITS[group][feature]` plus `log.warn('plan_limit_missing')`.
- Values change only through `npm run set-limit`, `set-setting`, `ai-off`/`ai-on` or the D1 console, and each change writes `updated_by` (`'script:<name>'` or `'console'`).
- A **new** key or limit row ships as a new custom migration with `INSERT OR IGNORE`. An existing value is never changed by a migration, so console edits are never overwritten.
- The `last_daily_run` key (Appendix A §9 fallback) is added only if that fallback is activated.

### 3.9 Events, consents, `ip_caps`, `llm_calls`

**`events`** (`services/events.ts`):
- `track(name, props)`: `props` is parsed with `EVENT_PROPS[name]`, a strict zod object; a failure throws, which is a programming error caught by tests. If the result is `{}`, `props` is stored as `NULL`.
- `country` comes from `request.cf.country` if it matches `^[A-Z]{2}$` and is not `XX` or `T1`, otherwise NULL.
- `campus_code` and `utm_*` are copied from the user row at write time. They are NULL for anonymous events and for `account_deleted`.
- `day = utcDay(now())`.
- One row per event. Client beacons are rate-capped (Part 04).
- Purged after 400 days through `events_created`.

**`consents`** (append-only):
- The latest row per `(user_id, kind)` wins, ordered by `created_at DESC, id DESC`.
- `ConsentState = Record<'terms'|'privacy'|'ai_processing', { version: string; granted: boolean; at: number } | null>`. A kind is "current" when `granted && version === LEGAL_VERSIONS[kind]`.
- `record()` inserts nothing when the latest row already has the same `version` and `granted`, so repeated calls cost no write.
- Only `ai_processing` can be withdrawn. Terms and privacy are withdrawn by deleting the account.

**`ip_caps`** (`services/abuse.ts`):
- The IP comes from the `CF-Connecting-IP` header.
- IPv4 is used as is. IPv6 is reduced to its /64 prefix: the first 4 hextets, expanded and lowercased, plus `::/64`. This prevents address rotation inside one allocation.
- `ip_hash` = first 32 hex characters of `HMAC-SHA256(IP_HASH_SECRET, day + ':' + ipKey)`.
- Atomic hit: `INSERT … VALUES (?day, ?hash, ?bucket, 1) ON CONFLICT (day, ip_hash, bucket) DO UPDATE SET count = count + 1 WHERE ip_caps.count < ?limit RETURNING count`. No row returned → `RATE_LIMITED`. A limit of `0` → `RATE_LIMITED` without SQL.
- Buckets:

| Bucket | Used by | Limit setting |
|---|---|---|
| `checker_ai` | `checkerTips` | `anon_checker_ip_daily_limit` (3) |
| `events_public` | `eventsPublic`, `meEvent` | `events_ip_daily_limit` (300) |
| `coupon_attempts` | `couponPreview`, `couponRedeem`, `checkout` with a coupon code | `coupon_ip_daily_limit` (20) |

- Rows are deleted when `day < utcDay(now() − 2 d)`; the primary key starts with `day`, so the purge is an index range delete.

**`llm_calls`**:
- One row per provider attempt, including failures (`ok = 0`). Calls that are blocked before the provider (consent, quota, budget) write no row.
- `error_code` is an `ErrorCode` or one of `http_4xx`, `http_5xx`, `timeout`, `network`.
- `cost_micro_usd` follows 3.7 (the estimate when usage is unknown).
- `user_id` is NULL when `anon = 1`.
- Purged after 400 days.

### 3.10 Coupons and redemptions

- **Code format.** `normalizeCouponCode(raw)` = `raw.trim().toUpperCase()` with spaces and hyphens removed. It must match `^[A-Z0-9]{4,20}$`, otherwise `COUPON_INVALID { reason: 'not_found' }`. `npm run coupon:create` generates codes of a 4–8 character college prefix plus 8 random characters from `A-Z2-9` without `I`/`O` (for example `DVIT7KQ3MZ9P`). A typed code is accepted only with `--code` and must be at least 10 characters.
- **Validity order** (first failure wins): `not_found` (missing or `active = 0`) → `expired` (`valid_until <= now`) → `plan_not_eligible` (`planId` not in `plan_ids`, or not an INR plan) → `already_used` (a `coupon_redemptions` row for (coupon, user) exists) → `exhausted` (`redemptions >= max_redemptions`) → `not_free` (only for `couponRedeem` when `percent_off < 100`).
- **Price.** `priceWithCoupon(list, pct) = pct === 100 ? 0 : max(100, round(list × (100 − pct) / 100))` paise. Razorpay's minimum order is ₹1.

**100%-off redemption** (`redeemFreeCoupon` → `grantPass({ provider: 'coupon' })`):

```sql
-- 1. claim a slot (single statement)
UPDATE coupons SET redemptions = redemptions + 1
 WHERE id = ?1 AND active = 1 AND valid_until > ?2 AND redemptions < max_redemptions;   -- changes = 0 → exhausted/expired
-- 2. one db.batch: INSERT coupon_redemptions (UNIQUE coupon_id,user_id) ; INSERT passes (provider_payment_id = 'coupon:<redemptionId>') ; INSERT events coupon_redeemed
-- 3. if the batch fails on the UNIQUE constraint: UPDATE coupons SET redemptions = redemptions - 1 WHERE id = ?1 AND redemptions > 0
--    and return the existing redemption's pass (idempotent success, see Part 04)
```

**Discounted paid coupon (Razorpay):**
- `checkout_intents.coupon_id` records the coupon when the order is created. Nothing is counted at that point.
- At grant time, the batch that inserts the pass also inserts `coupon_redemptions` with `INSERT OR IGNORE` and runs `UPDATE coupons SET redemptions = redemptions + 1 WHERE id = ?`. This UPDATE is unconditional: the customer has already paid, so an overrun is honoured and logged as `coupon_overrun`.

Coupons and redemptions are never deleted. Redemptions are pseudonymised on account deletion.

### 3.11 `webhook_events` dedupe

| Step | SQL / rule |
|---|---|
| Signature fails | Nothing is written. The event id is untrusted. Return 401. |
| Claim | `INSERT INTO webhook_events (provider, event_id, event_type, payment_id, outcome, received_at) VALUES (…, 'processing', now) ON CONFLICT (provider, event_id) DO NOTHING RETURNING event_id` |
| Already present, terminal outcome (`granted`, `revoked`, `duplicate`, `ignored`, `rejected`) | Return 200 with that outcome and do nothing else. |
| Already present with `processing` | Process again. This is safe because `grantPass`/`revokePass` are idempotent and emit side effects only on first effect. It covers a crash or timeout in the earlier attempt. |
| Finish | `UPDATE webhook_events SET outcome = ? WHERE provider = ? AND event_id = ?`. `duplicate` = verified event for an already-granted or already-revoked payment. `ignored` = an event type we do not act on, a partial refund, or an intent that no longer exists (account deleted: `log.error('payment_for_deleted_account')` → manual refund). `rejected` = verified signature but the re-fetch checks failed. |
| Transient error (D1, provider re-fetch) | Leave `processing` and return 500, so the provider retries. |
| Fallback key | If Razorpay omits `X-Razorpay-Event-Id`, use `event_id = '<payment_id>:<event_type>'` (Appendix A §9). |
| Retention | Purged after 90 days (`webhook_events_received`). Provider retries stop well before that: Dodo's last retry is about 10 h after the first attempt. |

### 3.12 Ids and timestamps

```ts
// src/server/core/ids.ts: RFC 9562 UUIDv7, lowercase
export function newId(): string {
  const b = new Uint8Array(16); crypto.getRandomValues(b);
  const ms = now();                                     // core/clock.ts, so tests are deterministic
  b[0] = Math.floor(ms / 2 ** 40) & 0xff; b[1] = Math.floor(ms / 2 ** 32) & 0xff;
  b[2] = (ms >>> 24) & 0xff; b[3] = (ms >>> 16) & 0xff; b[4] = (ms >>> 8) & 0xff; b[5] = ms & 0xff;
  b[6] = 0x70 | (b[6]! & 0x0f); b[8] = 0x80 | (b[8]! & 0x3f);
  const h = [...b].map((x) => x.toString(16).padStart(2, '0')).join('');
  return `${h.slice(0, 8)}-${h.slice(8, 12)}-${h.slice(12, 16)}-${h.slice(16, 20)}-${h.slice(20)}`;
}
export const pseudonymId = (): string => `del_${newId()}`;
// src/shared/schemas/common.ts
export const UuidV7Schema = z.string().regex(/^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/);
```

- Ids from the same millisecond are not ordered among themselves. Code that needs an order uses `created_at` first and the id only as a tie-break.
- `ids.test.ts` checks the format, that 10,000 ids are unique, and that ids sort by time across milliseconds.
- Times are `now()` epoch ms. `utcDay(ms)` → `'YYYY-MM-DD'`, `utcMonth(ms)` → `'YYYY-MM'`, and both are computed with integer math only (no `Date` in the data layer; `clock.ts` may use `Date` internally for formatting).

### 3.13 Migration policy (additive only)

| Allowed in a new migration file | Forbidden (fails `check:migrations`) |
|---|---|
| `CREATE TABLE`, `CREATE INDEX`, `CREATE UNIQUE INDEX` on new tables | `DROP …`, `ALTER TABLE … RENAME`, `ALTER TABLE … DROP COLUMN` |
| `ALTER TABLE … ADD COLUMN` that is nullable or has a constant default | drizzle's table rebuild pattern (`__new_<table>`, `INSERT INTO __new_`, `PRAGMA foreign_keys=OFF`) |
| `INSERT OR IGNORE` seed rows into `plan_limits` / `app_settings` | `UPDATE` / `DELETE` of existing rows; editing any file present in `origin/main` |

**Mechanics:**
- Generate with `npm run db:generate -- --name <snake_case>`. A custom SQL migration uses `drizzle-kit generate --custom --config drizzle.core.config.ts --name <name>`.
- Migrations apply **before** the code deploy (Appendix A deploy workflows), so the new code always sees the new columns.
- The previous code version, used on rollback, must tolerate added columns. It does, because every query names its columns, and `SELECT *` is forbidden by `check:boundaries`.

**Removing something** takes two steps: stop using it in code, then leave the column or table in place forever. If a value must disappear (for example a deprecated setting), the code ignores it.

**Frozen CHECK lists.** SQLite cannot change a CHECK constraint without rebuilding the table, so every enum CHECK in Appendix A is permanent. A new enum value needs a new column or table, which is why 3.2 puts no CHECK on growing value sets (`plan_id`, `feature`, `name`, `key`). See "Skeleton issues".

**Before production migrations:** `deploy-production.yml` saves a D1 Time Travel bookmark for both databases. Time Travel keeps 7 days on Free.

### 3.14 Retention

| Data | Kept | Removed by |
|---|---|---|
| `resumes`, `tailored_versions`, `cover_letters` | until the user deletes them or the account | user action / `eraseUserData` |
| `users`, `consents`, `quota_usage`, `checkout_intents` | life of the account | `eraseUserData` |
| `passes`, `coupon_redemptions` | at least 8 years (accounting), pseudonymised after deletion | never deleted in the MVP |
| `coupons`, `plan_limits`, `app_settings`, `ai_budget_daily`, `account_deletions` | indefinitely (no personal data; `account_deletions.clerk_user_id` is NULL once completed) | none |
| `events`, `llm_calls` | 400 days | `runDaily` |
| `webhook_events` | 90 days | `runDaily` |
| `ip_caps` | 2 days | `runDaily` |
| Encrypted nightly dumps | 14 days (GitHub artifact retention) | GitHub |
| D1 Time Travel | 7 days (Free) | Cloudflare |
| Browser IndexedDB drafts | until sign-out (`signOutAndClearDrafts`) or the next successful server save of that document | client |

The privacy policy states that deleted data can remain in encrypted backups for **up to 30 days**. That is a conservative upper bound over the 14-day dumps and the 7-day Time Travel window.

### 3.15 Account deletion: what happens to each table

`deleteAccount` (services/account.ts) → `eraseUserData(env, userId)` (data/account-data.ts):

| # | Database | Statement (every one filters on the bound user id) | Result |
|---|---|---|---|
| 0 | core | `INSERT INTO account_deletions (id, clerk_user_id, status, requested_at) VALUES (?, ?, 'started', now)` | audit row |
| 1 | docs | one `db.batch`: `DELETE FROM cover_letters WHERE user_id = ?` · `DELETE FROM tailored_versions WHERE user_id = ?` · `DELETE FROM resumes WHERE user_id = ?` | `docs_rows` = sum of `meta.changes` |
| 2 | core | one `db.batch`: | |
| | | `DELETE FROM consents WHERE user_id = ?` | deleted |
| | | `DELETE FROM quota_usage WHERE user_id = ?` | deleted |
| | | `DELETE FROM checkout_intents WHERE user_id = ?` | deleted (`passes.intent_id` may now point nowhere, which is fine) |
| | | `UPDATE events SET user_id = NULL WHERE user_id = ?` | anonymised; campus/utm/country stay for aggregates |
| | | `UPDATE llm_calls SET user_id = NULL WHERE user_id = ?` | anonymised |
| | | `UPDATE passes SET user_id = ?del, pseudonymised_at = now WHERE user_id = ?` | pseudonymised (one shared `?del = pseudonymId()`) |
| | | `UPDATE coupon_redemptions SET user_id = ?del WHERE user_id = ?` | pseudonymised (same `?del`) |
| | | `DELETE FROM users WHERE id = ?` | deleted (removes email, clerk id, attribution) |
| | | `INSERT INTO events (… name 'account_deleted', user_id NULL, props {"hadPass":…})` | audit event |
| | | `UPDATE account_deletions SET status = 'clerk_pending', docs_rows = ?, core_rows = ?, passes_pseudonymised = ? WHERE id = ?` | |
| 3 | Clerk | `deleteAuthUser(env, clerkUserId)` (a Clerk 404 counts as success) | success → `UPDATE account_deletions SET status = 'completed', clerk_user_id = NULL, completed_at = now` |
| 4 | Clerk failure | the row stays `clerk_pending`; `runDaily` retries step 3 every day; `log.error('clerk_delete_failed')` | |

Notes:
- The statement count is 1 + 3 + 10 + 1 = 15, plus the prelude, which stays under the 25-statement budget.
- Not touched because they hold no user id: `webhook_events` (payment ids only), `ip_caps` (daily-salted hashes), `ai_budget_daily`, `coupons`.
- Any active pass is lost. The UI warns before confirming. There is no automatic refund.
- `deletion-completeness.test.ts` seeds every table for two users, deletes one with a mocked Clerk that succeeds, then scans every column of every table in both databases. It asserts that none contains the deleted user's id, Clerk id or email, and that the other user's rows are unchanged. A second case with a failing Clerk mock asserts that the only remaining trace is `account_deletions.clerk_user_id`, and that the next `runDaily` clears it.

### 3.16 JSON export format (`meExport` → `exportUserData`)

Rule: **every column of every row linked to the user is exported, except `user_id`**:
- Keys are camelCase, matching the Drizzle keys.
- JSON text columns (`data`, `keywords`, `props`, `plan_ids`) are embedded as parsed JSON with plain `JSON.parse` (no zod on this path).
- Booleans are true/false.
- `export-completeness.test.ts` enforces the rule by comparing the column lists of the Drizzle tables with the keys of each export array.

```ts
type AccountExport = {
  exportFormat: 'account_export';
  exportVersion: 1;
  exportedAt: number;                       // epoch ms
  brandName: string;                        // BRAND_NAME
  profile: UserRow;                         // all users columns (id, clerkUserId, email, country, currency, campusCode, ft*, createdAt, lastSeenDay)
  authProfile: { firstName: string | null; lastName: string | null; emails: string[] } | null; // getAuthProfile; null on Clerk error
  consents: Omit<ConsentRow, 'userId'>[];
  passes: Omit<PassRow, 'userId'>[];
  checkoutIntents: Omit<IntentRow, 'userId'>[];
  couponRedemptions: (Omit<CouponRedemptionRow, 'userId'> & { couponCode: string })[];
  quotaUsage: Omit<QuotaUsageRow, 'userId'>[];
  events: (Omit<EventRow, 'userId' | 'props'> & { props: Record<string, unknown> | null })[];
  llmCalls: Omit<LlmCallRow, 'userId'>[];   // anon/ok are booleans via { mode: 'boolean' }
  resumes: (Omit<ResumeRow, 'userId' | 'data'> & { data: unknown })[];
  tailoredVersions: (Omit<TailoredRow, 'userId' | 'data' | 'keywords'> & { data: unknown; keywords: unknown | null })[];
  coverLetters: Omit<CoverLetterRow, 'userId'>[];
  warnings: ('auth_profile_unavailable')[];
};
```

- Statements: 1 user + 8 core selects + 3 docs selects. The Clerk profile is 1 subrequest. If it fails, the export still succeeds with `authProfile: null` and a warning.
- The browser saves the file as `exportFileName`-style `<slugified fullName or 'account'>-export-YYYY-MM-DD.json` through `saveBlob`.
- **CPU caveat:** a maximal paid account (20 masters + 100 tailored at 64 KB, about 8 MB) needs more than 10 ms CPU to parse and serialise. It is reachable only by paying users, and the ADR moves to Workers Paid at the first revenue. `exports.test.ts` keeps a "max account" fixture so the number is visible.

### 3.17 Skeleton additions (part 03)

| Name | Kind | Where | Why |
|---|---|---|---|
| `RESUME_LIMITS` | const | `src/shared/config/input-limits.ts` | One home for all per-field resume limits (3.4.1) |
| `BUILT_IN_SECTION_KEYS`, `SectionKeySchema`, `DateRangeSchema`, `BulletSchema` | consts/schemas | `src/shared/schemas/resume.ts` | Reused by api.ts and the editor |
| `resumeByteSize` | function | `src/shared/schemas/resume.ts` | The 64 KB check that maps to 413, outside zod |
| `KeywordSchema`, `JdKeywordsSchema`, `MatchResultSchema`, `HonestyFlagSchema` | schemas | `src/shared/schemas/resume.ts` | §7.3 types need runtime schemas for the cache column and API outputs |
| `<Singular>Row` types (18) | types | `src/server/data/schema/*.ts` | Typed rows (3.2) |
| `checkout_intents_status_created`, `webhook_events_received`, `events_created`, `coupon_redemptions_user` | indexes | `migrations/core/0000_init.sql` | Purges and lookups without full scans (3.3) |
| Extra CHECK constraints | constraints | both `0000_init.sql` | 3.3 |
| `coupon_attempts` | `ip_caps.bucket` value | core DDL CHECK | Coupon brute-force cap |
| `coupon_ip_daily_limit` | setting (int, default 20) | `app_settings`, `SettingsSchema`, seed | Limit for `coupon_attempts` |
| `ConsentState` | type | `src/shared/schemas/api.ts` | Latest consent per kind (3.9) |
| `AccountExport` | type + schema `AccountExportSchema` | `src/shared/schemas/api.ts` | `MeExportOutput = AccountExportSchema` |
| Statement counter | test helper | `tests/helpers/db.ts` | Enforces the 25-statement budget |
