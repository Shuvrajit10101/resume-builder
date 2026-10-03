# Simple SaaS Pick: AI Resume Builder (Rezi-style)

*Decision memo, 3 October 2026. Team: 1-3 people in India, zero capital, Claude Code subscription only.*

> Build an independent product with your own code, name, design and copy. Never use "Rezi", its templates or its text.

## Shortlist and scores

Eight simple products with $1M+ revenue were tested by a skeptical evaluator each (weights: simplicity 20%, zero-capital fit 20%, ease of getting customers 20%, revenue proof 15%, room vs free alternatives 15%, margin 10%).

| Product | Revenue evidence | Score | Verdict | MVP (weeks) | Likely MRR at month 12 |
|---|---|---|---|---|---|
| Rezi | STRIPE-VERIFIED (TrustMRR, re-fetched 2026-10-03): $251,264 MRR, $224,316 in the last 30 days, $10,260,842 all-time, 10,418 active subscript... | 6.6 | possible | 3.5 | $300-2,000 MRR, most likely around $800. Comparable TrustMRR resume tools cluster at $0-4K... |
| Senja / Testimonial.to | SENJA: The founders report $1M ARR themselves. Co-founder Olly Meakings posted on X on 17 Nov 2025: '$1,000,000 ARR', 3 years 9 months, 3,00... | 6.2 | possible | 3.5 | $300-1,500 MRR base case. $2-4K is an upside case only if a distribution wedge (WordPress ... |
| Chatbase | Confirmed. I re-fetched TrustMRR on 2026-10-03. It shows a Stripe API-verified MRR of $863,632 (about $10.4M ARR), $865,869 in the last 30 d... | 6 | possible | 4 | $300-2,000 MRR in the base case, $0-300 in the downside case, and about $5K only if the Wo... |
| Uniqode / QR Code Generator PRO | Category passes $1M; the big figures are old. (1) Egoditor (QR Code Generator PRO, now QRCG by Bitly): sales rose about 250% in two years to... | 6 | avoid | 3 | $0-1,500, base case about $300-800. The 2026 TrustMRR clones (StashKit, Scanely) are at $0... |
| Canny | FOUNDER-PUBLISHED: $3.4M ARR as of 28 March 2024 (Canny's own 'Year 7 in review' post), with 583 paying customers, 17 staff and $0 raised. T... | 5.5 | avoid | 4 | $100-$600 MRR is the likely range; $0 is a real possibility; $1,500+ would be an outlier. ... |
| Kaching Bundles | Founder-reported only. None of these figures are Stripe-verified or filed. (1) Starter Story breakdown, Jan 8 2025: Kaching Appz portfolio a... | 5.5 | avoid | 4 | $0-2,000 MRR in the base case for an unknown team entering a category of 472 apps with fre... |
| Polly | Founder-reported, not audited: co-founder Bilal Aijazi said Polly has 'multiple seven figures in ARR' with about 20 people and millions of M... | 5.1 | avoid | 3 | $200-1,500 (base case about $500). This assumes Marketplace approval lands in month 3-4. F... |
| Outrank | The revenue is reported by the founders and is not Stripe-verified. Growth partner Tibo posted $3M ARR (~$250-300K MRR) on X on Apr 28, 2026... | 5.1 | avoid | 4 | $0-2,500, with a likely median under $1,000. Stripe/Polar/Lemon Squeezy-verified clones on... |

## The pick

**Winner:** Rezi (rezi.ai), an AI resume builder that tailors resumes to job descriptions. Its revenue is Stripe-verified on TrustMRR (re-fetched 2026-10-03): $251,264 MRR (about $3.0M ARR), $224,316 in the last 30 days, 10,418 active subscriptions and $10,260,842 all-time. The company is US-based and was founded in 2015. We would build an independent, honest AI resume builder with job-description tailoring, sold as passes, with an India campus wedge and USD sales worldwide.

**Runner-up:** A Chatbase-style AI support chatbot trained on a business's own website and documents. Chatbase shows a Stripe-verified $863,632 MRR on TrustMRR (data last updated 2026-07-31), and Stripe's customer story says it reached $10M ARR in March 2026. We would sell it as a WordPress/WooCommerce plugin plus agency white-label at $19-49/mo, not as a cheaper Chatbase. Build it second, from month 4 or later, and only if the resume product hits a kill gate or plateaus. It reuses the same auth, billing, LLM-quota and abuse-limit code. Its advantages are higher ARPU and USD B2B sales from the start, using 'I trained a bot on your site' demo outreach (SiteGPT, a solo Indian founder, reports about $13-16K MRR from this playbook). Its costs are higher running cost, ongoing support on answer quality, and abuse risk from a public widget.

### Why it won

- Strongest revenue proof of any candidate. Re-fetched today, TrustMRR's Stripe API data shows $251,264 MRR, 10,418 active subscriptions and $10.26M all-time revenue. The founder separately posted about $3.2M ARR. Chatbase is the only other Stripe-verified candidate.
- Best fit with the 'simple' definition. The MVP takes about 3.5 weeks solo with Claude Code. It needs no platform approvals: Google sign-in with only email/profile scopes needs no CASA, and there is no marketplace review. PDF and DOCX are generated in the browser, so there are no headless-Chrome servers. There is no regulated data and no GPU.
- Lowest running cost and blast radius. About $20-70/mo at 100 paying users. LLM calls cost cents per user with quotas. There is no public widget that strangers can use to burn credits (Chatbase clone risk), no promise to keep redirects alive for years (QR risk), and no third-party-site embed support tail (Senja/Kaching risk).
- Self-serve, low-support consumer product. The user builds, downloads and leaves. Nothing has to be installed on a customer's website or theme.
- Large, visible price gap against the leader. Rezi charges $29/mo and $149 lifetime; we charge $9 for a 30-day pass and $49 lifetime. In India we charge ₹199 for 30 days, against ResumeGyani's ₹299 for 7 days.
- It is the only candidate where being an India-based team is a distribution advantage. The team can reach college, alumni and junior WhatsApp/Telegram groups and visit placement cells in person at zero cost. Rezi's own university Enterprise tier ($99/mo per 200 users) shows that institutions pay for this.
- Fastest path to first money. A manual ₹99 'send your resume + JD, get a tailored one-pager' offer can be sold in the team's own networks in week 1, before the software exists. This validates demand and builds a fresher-resume test set.

### Why not the others

- **Chatbase (runner-up):** It has the best revenue proof ($863K MRR, Stripe-verified) and higher B2B ARPU. It loses on simplicity and risk for a zero-capital solo team. Answer quality on messy real sites is an ongoing product, not a 4-week build. The public widget endpoint invites credit-burning abuse. Running cost is about $150-350/mo at 100 users, against $20-70 for resumes. Wrong-answer complaints create support load. Clones mostly earn $0-4K MRR (Chatwith $3.75K). Build it second, on the same stack.
- **Senja / Testimonial.to:** Third place. The $1M ARR is founder-reported, not Stripe-verified, and Testimonial.to's standalone figure is not cleanly confirmed. The 'cheaper' slot is already taken: Famewall and WiserReview at $9/mo, pay-once tools, AppSumo lifetime deals and generous free plans. Cross-browser video recording, uploads and playback are the hard engineering. TrustMRR clones from 2025-26 show $0-134 all-time, and the closest indie analog (Famewall) took about 19 months to reach $1K MRR.
- **Uniqode / QR Code Generator PRO:** Verdict 'avoid'. Free dynamic codes that never expire are already offered (Hovercode, ME-QR, QR-Verse), and paid plans start at $9-12. Phishing abuse can get the shared redirect domain blocklisted, which breaks every customer's printed codes at once. The team would also carry a multi-year uptime obligation. 2026 clones earn $0-150 a month.
- **Canny:** Verdict 'avoid'. Its $3.4M ARR (founder, 2024) is flat and comes from about 583 upmarket customers paying for Jira/Salesforce integrations, SSO and SOC 2, which a 4-week MVP cannot match. The cheap end is already free or $13-29 (Canny's own free plan, Featurebase, UserJot, Sleekplan, Fider).
- **Kaching Bundles (Shopify):** Verdict 'avoid'. Revenue is founder-reported only. The moat is fast live-chat support across a long tail of themes, which breaks the low-support rule. Billing must go through Shopify, not Dodo or Razorpay. The category has 472 apps, including free, 5.0-rated apps with 1,000-3,500 reviews.
- **Polly (Slack/Teams polls):** Verdict 'avoid'. Slack's API terms (from Oct 2025) forbid charging for a Slack app outside the Marketplace. Getting listed requires 10 active installs and up to about 12 weeks of review, which breaks the 'no slow approval' rule. Open Poll+ is free and unlimited, and Polly's seven figures come from enterprise and Teams sales.
- **Outrank (AI SEO autopilot):** Verdict 'avoid'. $3M ARR is founder-reported only. Google's 2026 spam updates were widely reported to hit mass AI content hardest, which is exactly what this product produces. Running cost per user is far higher (about $350-1,500/mo at 100 users). Verified clones earn $11 to $6.5K all-time. Growth came from the founder's audience, which this team does not have.

## What to build

An honest, India-first AI resume builder and job-description tailoring tool that matches Rezi's core feature set, built from scratch under the team's own original name, design and copy: structured editor, ATS-friendly single-column templates, real-text PDF and DOCX export, AI bullet/summary/cover-letter writing, and a job-description keyword match. The angle is pay-per-job-search passes with no auto-renew instead of a $29/mo subscription ($9 for 30 days, $49 lifetime; ₹99-₹999 by UPI). The second part of the angle is 'honest tailoring': the AI only rephrases what the user really did, shows a diff, and flags any number, employer, degree or skill that is not in the user's source data. For India it adds fresher and campus formats (one page, CGPA, projects, internships, certifications) and a per-cohort plan for college placement cells. The same product is sold worldwide in USD through Dodo Payments, and a free no-signup resume checker is the lead magnet for both markets.

### MVP features

- Sign-in by email magic link or Google sign-in using only basic email/profile scopes (no CASA). Supabase Auth with row-level security so each user sees only their own resumes.
- Structured resume editor built on one JSON schema: contact, summary, education (with CGPA/percentage), experience, projects, internships, skills, certifications and achievements. Sections can be dragged to reorder, with autosave and a live paginated preview.
- 6 original single-column templates: 3 US/global formats and 3 India fresher/campus one-page formats. They are our own designs, not copies of Rezi's.
- In-browser export: PDF with a real selectable text layer and embedded fonts (@react-pdf/renderer or Typst-WASM), plus DOCX (the docx npm library). No server-side rendering.
- Import of an existing resume from PDF or DOCX: pdf.js or mammoth extracts the text in the browser, a cheap LLM maps it into the JSON schema, and the user confirms each section.
- AI bullet rewriter, summary generator and cover-letter generator with 'honest tailoring' guardrails. A side-by-side diff is shown, and an automatic check highlights any numeral, company, degree or skill in the output that does not appear in the user's source data. Nothing is applied without the user accepting it.
- Job-description tailoring: paste a job description, then extract hard skills and keywords, show a transparent keyword-match % (with the formula visible), list missing keywords, and offer one-click 'save as tailored version'. Multiple versions per user, one per job.
- Rule-based content checks with no LLM cost: weak verbs, bullets without numbers, first-person pronouns, length and page overflow, date and tense consistency, missing contact fields.
- A free no-signup resume checker as a static page: upload a PDF and get a keyword or structure report with the top 5 fixes. Protected by Cloudflare Turnstile and a per-IP daily cap, and it ends with a 'fix these in the editor' call to action.
- Paywall and entitlements. The free tier allows unlimited editing, the checker, a small daily AI quota and 1 PDF download. Passes come with an expiry date and no auto-renew: Dodo Payments checkout plus webhook for USD, Razorpay Orders plus webhook for INR. Lifetime comes with a published fair-use AI cap, and campus coupon codes are applied manually.
- Per-user and per-plan LLM quotas enforced in one edge-function proxy (the only place API keys live), with logging of tokens per user so unit costs are visible from day one.
- Privacy basics required for resume PII: consent at signup, data export (JSON), one-click account deletion, a privacy policy and terms (DPDP Act, GDPR-aware), and a paid-tier LLM API whose terms say prompts are not used for training.

### Skip for version 1

- AI mock interviews (voice or video) and any audio or video processing
- Job tracker, auto-apply bots and any LinkedIn import or scraping (it breaks LinkedIn's terms)
- Chrome extension
- Self-serve college/coach admin dashboard: run the first cohorts with coupon codes and a spreadsheet
- Recurring subscriptions in India (UPI AutoPay or e-mandates): sell prepaid passes only
- USD auto-renewing subscriptions: passes and lifetime only for now (avoids Dodo's +0.5% subscription fee and auto-renewal-law exposure)
- More than 6 templates, a template marketplace and a custom template designer
- Hindi or regional-language UI (offer Hindi/Hinglish help over WhatsApp manually first)
- Programmatic SEO at scale. Hand-write about 10-20 pages first and expand only the ones that rank.
- Affiliate software and dashboards: track a few creator coupon codes manually
- Native mobile apps (make the web editor mobile-usable instead)
- Human resume-writing service as a product line (only the manual week-1 validation offer)
- AppSumo or other lifetime-deal marketplaces
- Team seats, SSO and enterprise contracts

## Pricing

- **Global (USD):** Free: unlimited editing, the free checker, 3 AI rewrites a day and 1 PDF download. Job Search Pass: $9 one-time for 30 days, unlimited downloads, DOCX, tailoring and cover letters, with no auto-renew. Season Pass: $19 for 90 days. Lifetime: $49 with a published fair-use AI cap, sold as 'founder pricing' to the first 200 buyers. Do not go below $9: Dodo charges 4% + 40c per US card transaction and +1.5% on non-US cards (verified on Dodo's pricing page today), so a $9 sale nets about $8.10-8.25.
- **India (INR):** Free tier as above. ₹99 for 7 days, ₹199 for 30 days, ₹499 for a 90-day 'placement season' and ₹999 lifetime (fair use), all paid one-time by UPI or card through Razorpay (about 2% fee). Campus cohort plan: about ₹25 per student per season with a ₹5,000 minimum per college, after one free pilot season. Prices are GST-inclusive in the copy. Check with a CA when to register for GST; the general services threshold is ₹20 lakh turnover.
- **Versus the incumbent:** Rezi: Pro $29/mo, Lifetime $149, Enterprise $99/mo per 200 users. Ours: $9 for 30 days (69% below one month of Rezi Pro, with no renewal), $49 lifetime (67% below), and a campus plan far below Rezi Enterprise for Indian colleges. Other references: Kickresume $24/mo monthly, UseResume $11 for 7 days, ResumeGyani ₹299 for 7 days or ₹1,499/yr. Be honest about what this means: we are cheaper than Rezi but not cheaper than free (Reactive Resume, Google Docs/Overleaf templates, ChatGPT/Gemini/Claude free tiers). Price is a reason to pick us over Rezi, not over free tools; the honest-tailoring workflow and the campus formats have to do that work.

## Free-tier tech stack

Everything below costs $0 until revenue, except a domain and a small prepaid LLM credit. Note: the Claude Code subscription does not include API credits.

- Marketing site, free checker and SEO pages: Astro static site on Cloudflare Pages (free plan, commercial use allowed). Do not use Vercel Hobby; its terms are non-commercial.
- App: Vite + React + TypeScript single-page app on Cloudflare Pages, with Tailwind and shadcn/ui for the editor.
- Backend: Supabase free tier for Postgres, Auth and row-level security. Supabase Edge Functions act as the LLM proxy and the payment-webhook receiver. Free projects pause after about a week of inactivity, so upgrade to Supabase Pro ($25/mo) once revenue passes about $100/mo; this also adds backups.
- Documents: @react-pdf/renderer (or Typst compiled to WASM) for real-text PDFs in the browser, the docx npm library for DOCX, pdfjs-dist and mammoth for import. No headless Chrome, no server cost.
- LLM: a cheap paid-tier 'flash/haiku/mini'-class model behind the edge-function proxy, with per-user quotas. Do NOT ship on the Gemini free tier: Google's terms say free-tier prompts and responses are used to improve Google products and may be read by human reviewers (outside EEA/UK/CH), which is unacceptable for resume PII. Start with a $5-10 prepaid credit; expect about $10-40/mo at 100 paying users.
- Anti-abuse: Cloudflare Turnstile (free) and per-IP and per-user rate limits.
- Email: Resend free tier for magic links and receipts.
- Analytics and errors: Cloudflare Web Analytics (free), PostHog free tier for the signup-to-paid funnel, Sentry free developer tier.
- Payments: Dodo Payments as merchant of record for USD, with no setup or monthly fee. Fees are 4% + 40c on US cards, +1.5% outside the US and +0.5% on subscriptions. Payouts under $1,000 carry a $5 fee and USD SWIFT payouts for non-US businesses cost $25, so batch payouts. Razorpay for INR (about 2%, no setup fee, KYC required). Start both KYC processes in week 1.

Unavoidable cash: about $10-15/yr for a .com domain (a .in is optional) plus $5-10 of LLM credit, so about $20-25 before the first sale. After revenue: Supabase Pro $25/mo (optional at first) and per-transaction payment fees.

## First 30 days

1. Week 1 (Oct 5-11): Pick an original brand name and clear it on IP India and USPTO trademark search and for domain availability, then buy the domain. Apply for Dodo Payments and Razorpay KYC now, because review takes days. Write the privacy policy and terms. Set up the repo, Supabase schema and row-level security, sign-in, the editor skeleton, 2 templates and in-browser PDF export. In parallel (second founder, or evenings): post a manual offer in 5-10 of your own college, alumni and junior WhatsApp/Telegram groups: 'Send your resume + a job description, get a tailored one-page resume in 24h, ₹99.' Do the work with Claude, collect payment by UPI, and log every request. Goal: 10+ paid orders, 20+ real fresher resumes (with consent) to use as a test set, and notes on what students actually ask for.
2. Week 2 (Oct 12-18): PDF/DOCX import with LLM structuring; AI bullet rewriter, summary and cover letter with the diff view and the 'not in your source data' highlighter; job-description keyword extraction and match %; rule-based content checks; DOCX export; templates 3-6 (3 global, 3 India fresher/campus). Test every template's PDF by copy-pasting its text and running it through a free open-source resume parser to confirm the text layer reads in order.
3. Week 3 (Oct 19-25): Free no-signup checker page with Turnstile and per-IP caps; plans, paywall and entitlements with expiry dates; Dodo checkout and webhook (USD); Razorpay order and webhook (INR); LLM quota proxy with token logging; data export and account deletion; error tracking and funnel analytics. Write 5 honest pages (e.g. 'fresher resume format for campus placements', 'how to read a job description for keywords') and 1 'how our keyword match is calculated' page.
4. Week 4 (Oct 26-Nov 1): Closed beta. Give free 30-day passes to the week-1 customers plus 20-30 final-year students and ask each for a 10-minute call. Fix the top 10 issues. Soft launch: post the free checker in your own groups; start a daily anonymized 'resume teardown' post on LinkedIn (before/after, with consent); answer 3-5 resume questions a day on r/resumes, r/developersIndia and r/cscareerquestionsIN within each subreddit's rules (link on the profile, not in comments). Email or visit 10 placement cells or training institutes offering a free pilot for one batch. Submit the free checker to free AI-tool directories.
5. Days 29-30: Review the numbers against gate 1 of the kill criteria: checker uses, checker-to-signup %, signup-to-paid %, paid passes by currency, LLM and payment cost as % of revenue, and top refund or complaint reasons. Choose one change (price, onboarding or the checker's call to action) for the next 30 days. Schedule a Product Hunt launch for week 6-8, once the product is polished and has a few testimonials.

## Getting the first customers

FIRST 10 (weeks 1-4, all zero cost). (1) Pre-sell the manual ₹99 tailoring service in the team's own college, alumni and junior WhatsApp/Telegram groups during the current placement and internship season. Each buyer later gets a free 30-day pass in the product and is asked for a testimonial. (2) Turn the best 3 before/after resumes into anonymized LinkedIn teardown posts with the buyers' consent. (3) Ask every beta user to forward the free checker to their batch group. Goal: 10 paying customers, even if they are ₹99 ones.

FIRST 100 (months 2-4). (a) Campus: run free one-batch pilots with 2-3 placement cells or training institutes the team can visit in person. Measure usage, then quote the cohort plan (about ₹25 per student per season, ₹5,000 minimum). One converted college can bring 100-500 students. (b) Creators: offer Indian placement-prep and career YouTubers, LinkedIn creators and job-alert Telegram channels a 30% revenue share on their own coupon code, tracked manually and paid only after sales. (c) Global USD: treat the free checker and the 'how keyword match works' page as the main hook. Publish short teardown videos (YouTube Shorts and LinkedIn) aimed at US and other English-speaking job seekers. Answer helpfully in r/resumes, r/jobs and r/cscareerquestions within the rules. Launch on Product Hunt in week 6-8 and list on free directories (AlternativeTo, SaaSHub, AI tool lists). (d) SEO: about 20 hand-written long-tail pages (role + fresher/India, role + resume keywords), expanding only those that get impressions in Google Search Console after 6-8 weeks. Avoid mass-generated pages; Google's scaled-content-abuse policy targets them.

Do not: buy ads, spam groups or DMs, use the word 'Rezi' in names or ads, or promise 'beat the ATS'. Team split: one person builds, one person sells and posts every day. If you are a solo founder, cap building at 50% of your time after week 4.

## Kill criteria

Gate 0 (day 14): fewer than 10 people in the team's own networks pay ₹99 for manual tailoring. That means the India channel is weak; do not build the campus features, ship a global-only MVP and re-test.

Gate 1 (day 30, about launch): fewer than 10 paid passes in total, or checker-to-signup below 10%. Change one thing (the checker's call to action, price or onboarding) and re-measure for 30 days.

Gate 2 (day 90): trailing-30-day revenue below $300 equivalent (USD + INR converted), signup-to-paid below 1% after 1,000 signups, and no college paying for a cohort. Stop feature work, leave the product running on free tiers (about $0-25/mo), and start the runner-up (WordPress/agency AI chatbot) on the same stack.

Gate 3 (day 180): trailing-30-day revenue below $800 and not growing month over month. Put it in maintenance mode or sell it on a micro-acquisition marketplace.

Hard stops at any time: LLM + infra + payment costs above 30% of revenue after quotas are tightened; a trademark or deceptive-claims complaint that cannot be resolved by renaming or rewording; or Dodo/Razorpay account holds caused by refunds above 5%.

## Honest risks

- The code is not the hard part; distribution is. Rezi's lead comes from about 10 years of SEO (around a third of the Google top 10 for 'AI resume builder' in 2024). A new domain starts at zero, and the head terms belong to Zety, Resume.io, Canva and Indeed. Resume clones on TrustMRR mostly earn $0-4K MRR.
- The category leader is shrinking. Rezi went from $288K MRR (about Feb 2026) to $251,264 today (about -13%), was listed for sale with the price cut from $23M to $15M, and Similarweb showed traffic down 8% month over month in Jun 2026. Free chatbots (ChatGPT, Gemini, Claude) now do resume tailoring at no cost.
- Revenue is built to churn. Job seekers leave once hired, and free-to-paid conversion in this category can be very low (ResuMax about 0.3% to subscriptions). Most revenue will be one-time passes and lifetime deals, not durable MRR.
- Indian users pay little. Tickets are ₹99-499, ResumeGyani already sells passes, Reactive Resume is free, and student platforms may already bundle free builders. Many hundreds of paid passes are needed to reach even $500/month.
- Privacy and LLM terms. Resumes are personal data under India's DPDP Act and GDPR. The Gemini free tier lets Google use prompts to improve its products and allows human review (outside EEA/UK/CH), so production must use a paid API. Consent, export and deletion must work from day one.
- Deceptive-claims and fabrication risk. 'ATS score' is mostly a marketing construct. Avoid 'beat the ATS', 'ATS-approved' or 'guaranteed interviews' (FTC in the US, ASCI in India). If the AI invents credentials, users can be harmed and the brand damaged, so the source-data highlighter is not optional.
- Trademark and trade dress. Never use 'Rezi', 'Rezi Score', their template looks or their copy, in the product or in ads. Factual comparison pages are the limit.
- Channel bans. Reddit, LinkedIn and WhatsApp groups punish self-promotion. One spammy week can burn the team's own networks, which are the main zero-cost channel.
- Payment friction. Merchant-of-record KYC and website review can take days. Dodo payouts under $1,000 carry a $5 fee and USD SWIFT payouts cost $25. Razorpay KYC is required, and GST becomes relevant as revenue grows. Plan for 2-4 weeks between the first sale and money in the bank.
- Seasonality and timing. Indian campus hiring is seasonal and US hiring is cyclical, so expect lumpy months that can be misread as product failure or success.
- SEO policy risk. Google's scaled-content-abuse policy and AI Overviews limit the programmatic 'resume for [role]' pages that grew Rezi.

## Realistic outcome

Base case: a working, sellable product in about 4 weeks for about $20-25 of cash. 10-30 paid passes in the first month after launch, mostly ₹99-199 (roughly $25-100 total). $150-500/month by month 3. $300-2,000/month by month 12, most likely around $800/month, mostly from one-time passes and lifetime sales rather than recurring MRR. Upside of about $3-5K/month comes only if 3-10 colleges or institutes buy cohort plans and a few creators drive coupon sales. Downside is under $100/month if neither the campus channel nor the free checker gets traction. This is consistent with verified clones (Eztrackr about $4K, LetMeApply $2.6K, ResuMax $1.9K MRR); reaching Rezi's $3M ARR is not a realistic 12-month goal. Running cost stays under about $70/month, so the financial downside is near zero; the real cost is 3-6 months of founder time. Even in the base case the team ends up with a reusable stack (auth, MoR + Razorpay billing, LLM quota proxy, abuse limits) and distribution experience, which makes the runner-up chatbot product faster to ship. Note: this pick follows the instruction to drop complex products and differs from the 2026-10-02 report's first-build pick (COD/RTO attribution for D2C).

## Sources (evaluator citations)

- https://trustmrr.com/startup/rezi
- https://x.com/jacob_jacquet/status/2100183748583309696
- https://x.com/marclou/status/2025848118806532283
- https://www.threads.com/@marclouvion/post/DVF-7uqlDJx/highest-mrr-startup-on-trust-mrr-just-dropped-their-sale-price-m-m-ai-resume
- https://atticusli.com/startup/rezi/
- https://www.rezi.ai/pricing
- https://www.similarweb.com/website/rezi.ai/
- https://www.starterstory.com/stories/ai-powered-free-ats-resume-builder
- https://growthpartners.online/stories/how-rezi-grew-a-25m-arr
- https://trustmrr.com/category/recruiting
- https://trustmrr.com/startup/resumax-ai
- https://trustmrr.com/startup/ai-resume-builder-rothr
- https://trustmrr.com/startup/rezly-ai
- https://trustmrr.com/startup/applygigs
- https://github.com/amruthpillai/reactive-resume
- https://resumeoptimizerpro.com/blog/free-resume-builder-reddit
- https://resumegyani.in/
- https://www.kickresume.com/en/help-center/best-ai-resume-builders/
- https://appsumo.com/products/resumeupai/
- https://www.producthunt.com/categories/resumes
- https://www.wipperoz.com/en/blog/ai-resume-builders-trend-hunter-market-signal
- https://www.thesuccessfulprojects.com/how-two-indie-hackers-built-a-successful-micro-saas-senja-io-1m-arr/
- https://jobs.senja.io/
- https://senja.io/pricing
- https://testimonial.to/pricing
- https://the-seo-autopilot.com/en/case-studies/damon-chen
- https://supabird.io/articles/from-corporate-dreams-to-1-3m-arr-as-an-entrepreneur
- https://www.ideaindex.so/case-studies/testimonial-to
- https://getlatka.com/companies/senja.io#funding
- https://famewall.io/founder-stories/journey-to-1000-mrr/
- https://www.startups.fyi/product/famewall
- https://trustmrr.com/startup/endors
- https://trustmrr.com/startup/saywall
- https://wiserreview.com/blog/senja-alternatives/
- https://getcredibly.org/blog/testimonial-to-alternatives
- https://sayabout.us/blog/senja-alternatives
- https://testimonials.ltd/blog/senja-vs-testimonial-to-pay-once
- https://appsumo.com/products/feedspace/
- https://appsumo.com/products/testimonial-hunt/
- https://appsumo.com/products/social-testimony/
- https://shapo.io/
- https://github.com/arpanp001/testimonial-collector
- https://github.com/PavithraNagineni/Testimonial-Social-Proof-Collector
- https://trustmrr.com/startup/chatbase
- https://stripe.com/customers/chatbase
- https://getlatka.com/companies/chatbase.co
- https://www.chatbase.co/pricing
- https://productled.com/blog/how-chatbase-hit-8m-arr-with-18-people
- https://trustmrr.com/startup/chatwith
- https://trustmrr.com/country/us
- https://trustmrr.com/category/ai
- https://superframeworks.com/case-study/sitegpt
- https://founderspec.com/breakdowns/sitegpt
- https://www.indiehackers.com/post/chatbase-just-crossed-10m-arr-bootstrapped-here-is-a-teardown-of-a-5-8k-mrr-mini-chatbase-hidden-on-the-market-83e52469c7
- https://openalternative.co/alternatives/chatbase
- https://sitespeak.ai/blog/best-chatbase-alternatives
- https://appsumo.com/products/charla/
- https://appsumo.com/products/tiny-talk/
- https://appsumo.com/products/chatwith/
- https://appsumo.com/products/chatnode-ai/
