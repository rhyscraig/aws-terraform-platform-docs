# Portfolio Hosting Architecture — craighoad.com + terrorgems.com

**Status**: Design — three adversarial review rounds complete, now mid-execution
**Date**: 2026-08-09 (v1) → v2 (round-1 fixes) → v3 (round-2 fixes) → v4 (round-3 fixes) → v5
(2026-08-12, execution-progress correction, this revision)
**Owner**: Platform
**Supersedes**: Ad-hoc assumptions in old planning docs (see §7)

---

## 1. Purpose

Define the target-state hosting architecture for craighoad.com (+ blog) and terrorgems.com
(+ blog), and the measured plan to reach it. Zero technical debt, cost accurately understood and
controlled, easy to manage.

**Revision history, condensed** (full record in §8): round 1 found a materially false claim
(craighoad-blog "never deployed" — it is) and an incomplete debt inventory. Round 2 found several
round-1 fixes were cosmetic or wrong, most notably an orphaned "Stage 1b" this document introduced
but never actually folded into the companion roadmap's numbered sequence. Round 3 found the WAF
and tax figures round 2 corrected were themselves still off, and confirmed a real IAM logic bug in
the tag-enforcement SCP design (see the roadmap doc §8 for full detail — the SCP design lives
there, not here). v4 fixed what round 3 found and was handed off for execution.

**v5 (this revision) is not a fourth adversarial round — it's a correction against ground truth
from actual execution.** A real engineering session picked up v4 and built most of Stage 3 (the
tag-enforcement guardrail) and confirmed Stage 5's audit finding — and in doing so, found: this
document's own file had never been committed to git (fixed as part of this revision); one of the
three "dead" repos in §3.1–3.6 turns out to have live commits and its own Terraform (§3.1–3.6,
corrected below); a production-breaking bug in craighoad-blog unrelated to anything either plan
anticipated; and a real tag-casing bug in the SCP that risks blocking the next legitimate deploy.
See §9 (new) for the full execution-progress summary. Treat this version as authoritative — it
supersedes v4 on every point where live execution contradicted the design.

---

## 2. Current State (verified live 2026-08-09; corrected v2; re-verified v3 — all v2 claims held up under independent re-check)

### 2.1 craighoad.com (account: `hcp-craighoad-prod`, 624426145233)

| Resource | Value |
|---|---|
| Hosted zone | `craighoad.com` (Route53, in-account) |
| CloudFront | `E5ICAEFHQWU4Y`, aliases `www.craighoad.com` and `craighoad.com` (apex), Deployed |
| Origin | S3 `www.craighoad.com` |
| Status | Live, HTTP 200 |
| Repo (frontend) | `craighoad-portfolio-website` — React 19 + Vite, actively maintained |

craighoad-blog is deployed (not pending — round-1 correction, confirmed accurate again on
independent round-2 re-check): Lambda `hcp-blog-api` + API Gateway `fhu7jtpt1l` (HTTP API) +
DynamoDB `hcp-blog-posts`, live in **`eu-west-1`**. Whether it's wired into the live site is
resolved in §5 Stage 5 (see the roadmap doc — this is now a real, numbered stage, not the orphaned
"Stage 1b" reference v2 left dangling; see §9).

Also present, out of scope: `ai.craighoad.com` (separate CloudFront `E32AA1MMVIZN2F`, its own
Lambda@Edge function). Cost not cleanly separable from craighoad.com's without cost-allocation
tags — see §2.4.

### 2.2 terrorgems.com (account: `hcp-terrorgems-prod`, 767828739298)

| Resource | Value |
|---|---|
| Hosted zone | `terrorgems.com`, MX configured; **6 DKIM CNAME records specifically** (the zone has 8 CNAMEs total including 2 unrelated ACM validation records — "6" refers only to the DKIM set, clarified in v3 to avoid misreading), only 3 currently active/signing |
| CloudFront | `E19Y75BSQTMF08`, aliases `www.terrorgems.com` and `terrorgems.com` (apex), Deployed |
| Origin | S3 `terrorgem-prd-frontend-767828739298` |
| API | Lambda `terrorgem-prd-api` behind API Gateway HTTP API |
| Data | 31 product DynamoDB tables (32 including the Terraform lock table) |
| Status | Live, HTTP 200 |

TerrorGems' blog stays a feature module inside the platform. **Round-2 finding on this reasoning**:
the named risk mitigations (Lambda aliases/versioning for safer deploys, reserved concurrency if
the blog-agent cron and interactive traffic contend) were never actually scheduled anywhere in
either document — naming a risk isn't the same as instrumenting or scheduling a response to it,
which is exactly the standard §4.7's Lambda-warmer fix holds itself to. Corrected in §4.2 below:
either schedule these concretely, or state honestly that they're accepted-but-unmonitored risks,
not implied-to-be-handled ones.

SES sandboxed, Stripe webhook secret empty — see the roadmap doc's §0 (urgent, independent of any
phased plan).

### 2.3 Account/DNS boundary

Per-account DNS kept. **Round-2 finding**: v2's justification ("day-to-day operability win
outweighs the DR scenario's likelihood") was two-sided hand-waving instead of one-sided — an
improvement, but still an unquantified assertion on both sides, and "quarantine an account" was
never defined procedurally. **Honest correction rather than a false fix**: this tradeoff genuinely
isn't quantifiable with the information available (no incident history to estimate likelihood
from, no defined quarantine runbook to time). Stated plainly: **this is a judgment call, not a
calculation** — for a solo operator at near-zero current traffic, the simplicity of one DNS zone
per account is chosen deliberately, accepting that a future account-level security incident would
also compromise DNS control for that domain. If a quarantine runbook is ever written (out of scope
here), revisit this decision against it. Don't dress up an unquantifiable judgment call as a
weighed tradeoff with numbers that don't exist.

### 2.4 Current cost run-rate — v3: corrected again, the v2 figures didn't reconcile with themselves

**Round-2 finding**: v2's own cost table didn't add up. It stated terrorgems-prod's run-rate as
"~$7.80/mo" but its own itemized drivers (KMS 2.53 + CloudTrail 1.20 + SecretsMgr 1.03 + Route53
0.50 + cents) summed to ~$5.36 — a $2.44 gap. Traced to two errors: (1) **Tax/VAT (~19.4%,
confirmed from this account's real invoices) was priced nowhere in v1 or v2**, and applies to the
whole bill; (2) Route53's flat $0.50/mo hosted-zone fee was linearly extrapolated from a 9-day
window instead of treated as the lump-sum charge it actually is, inflating the figure. Both fixed:

| Account | Corrected monthly run-rate | Drivers |
|---|---|---|
| terrorgems-prod | ~$6.63/mo pre-cleanup | KMS $2.53 (2 orphaned keys, not 1 — see roadmap §3.7), CloudTrail $1.20 (orphaned trail, still actively logging), Secrets Manager $1.03 (2 of 3 secrets orphaned), Route53 $0.50, S3/DynamoDB/API GW/Lambda: cents, **plus ~19.4% tax on all of it** |
| craighoad-prod | ~$2.43/mo | Includes ai.craighoad.com's cost, not cleanly separable — see below |
| management | ~$2.34/mo | KMS + Secrets Manager, heavier API-call volume than the flat rate implies |
| log-archive | ~$0.73/mo | matches |
| **Total** | **~$13.30/mo current** (this figure held up under round-2 re-verification) | |

**Corrected line items** (v2 additions, all independently re-confirmed accurate in round 2):

- KMS: `alias/terrorgem-prd-dynamodb` (legitimate, cost scales with DynamoDB traffic, not flat)
  and `alias/asatst-prod-s3-key` (orphaned). **Round-2 addition: a third, previously-unfound
  orphaned KMS key** exists (`dd804b22-...`, no alias, unused, identical description to the
  legitimate key — likely mistaken for it during any casual audit). See roadmap doc §3.7 for full
  detail; this key was not in v2's cleanup checklist and would have survived cleanup untouched.
- Secrets Manager: live cost today, not future-only. **Round-2 correction**: 2 orphaned secrets
  cost $0.80/mo at the confirmed $0.40/secret rate, not the $0.27/mo v2 stated (a 3x
  understatement, corrected in the roadmap doc's §3.7).
- WAFv2: **round-3 correction — was $7.00/mo, actually $8.00/mo.** Re-checked against real
  historical billing: the base $5.00/mo WAFv2 charge in this account ran continuously from at
  least January 2026 through late July 2026 (six-plus months, not a short July "pilot" as v3
  described), and its rule-group charge totals exactly $3.00/mo (3 rule-group-units), not $2.00/mo
  — so the real historical configuration was 3 managed rule groups, not 2.
- API Gateway HTTP API free tier: confirmed expired for this account (joined the org 2024-09-09,
  a 12-month new-account perk).
- ai.craighoad.com cost bleed: still unresolved — add cost-allocation tags to separate it (§6).
- Tax/VAT: **round-3 correction — v3 used ~19.4%, read from the current, still-estimating partial
  month.** Three consecutive *finalized* monthly invoices show ~20.0% consistently — use that
  figure going forward.

**CloudFront `PriceClass_100` vs. India** — unchanged conflict. An *unapplied* Terraform
environment for `terrorgems.in` exists in the terrorgems-platform repo (`ap-south-1`,
`PriceClass_200`, **INR-primary with GBP fallback pricing** — round-3 correction: v3 called this
"INR-only," which the actual Terraform variables don't support; both currencies are configured).
`PriceClass_200` still doesn't include India edge locations (only `PriceClass_All` does), so it
doesn't resolve the conflict, but whoever picks this up should know the work exists.

**Corrected target budget — round 3, moved again**: v2 said ~$10–12/mo, v3 corrected to ~$14/mo
using a WAF estimate and tax rate both since found to be understated. With the corrected WAF
($8.00) and tax (~20.0%) figures above (full derivation in the roadmap doc's §5): **~$15/mo**.
This figure has moved upward in every round as more real billing data was checked — treat it as
the most-verified figure so far, not necessarily final. Per this account's own project
documentation, `hcp-terrorgems-prod` is an **explicitly accepted exception** to the platform's
$0/month goal — this isn't a demand for a lower number, it's the real one. Budget alert threshold:
keep $20 (roadmap doc §5), now ~33% headroom over the corrected target, tighter than either prior
round implied.

---

## 3. Technical Debt Inventory

### 3.1–3.6: **v5 correction — `website-react-portfolio-craighoad.com` is not dead, remove it from the archive list**

Previously listed as a dead repo alongside two genuinely dead ones. **This was wrong.** Confirmed
live during execution (2026-08-12): this repo has commits as recent as `2026-08-11T20:57:07Z`,
including its own live Terraform (S3 + CloudFront via `terraform-aws-modules`), its own
`deploy.yaml`, and it independently picked up the same Project-tag-prerequisite fix craighoad-blog
needed for Stage 3 (commit `48f6ce4`). Do not archive it. Re-examine what it actually is (a
superseded predecessor to `craighoad-portfolio-website`? A parallel live thing? Unclear — worth a
short investigation before any archival decision) rather than trusting the original classification.

Remaining genuinely dead repos: `azure-terraform-solutions-terrorgem-platform` (confirmed 0 live
Azure resources) and `gcp-terraform-solutions-terrorgem-platform` (GCP check still genuinely open,
§6 — still not archived, correctly).

Other 3.1–3.6 items, still accurate: duplicate empty API Gateway; three overlapping budgets; the
unprefixed `TerrorGems` DynamoDB table (do not delete without the PITR + CloudTrail-lookback
verification — now with an honest limit on what that verification can prove, see §3.7 below and
the roadmap doc's Stage 1); `hcp-qa` stray buckets.

### 3.7 Contamination — see the roadmap document's §3.7 for the authoritative, current inventory

This section now lives primarily in the companion roadmap document, since it's entirely within
`hcp-terrorgems-prod` and Stage-by-stage remediation is tracked there. Summary for this document's
purposes: contamination is broader than v1 found (a 5th orphaned EventBridge rule, a second
actively-logging CloudTrail trail, **two** orphaned KMS keys as of round 2 — not one, orphaned
secrets, and SSM contamination running in both directions, including craighoad.com's own config
sitting inside the TerrorGems account). **Round-2 correction to v2's forensics plan**: v2 said the
account's CloudTrail history should be searched for "the originating principal" before deleting
the orphaned bucket. This is now proven impossible, not just difficult — the orphaned KMS key
predates the orphaned trail's own logging start by 4 minutes, and the org-level trail didn't exist
until 10 months later. No CloudTrail source in this account was ever capable of recording the
origin. The roadmap doc's Stage 1 acceptance criteria are corrected to stop promising this.

---

## 4. Target Architecture

### 4.1 One pattern, two instantiations — unchanged, held up under both review rounds

Static/SPA hosting (S3 + OAC + CloudFront + optional WAF) and the content/API microservice
pattern (API Gateway + Lambda + DynamoDB) remain the two consistent primitives.

### 4.2 TerrorGems' blog stays in-platform — round-2 fix: the mitigation is now honest about being unscheduled

**Round-2 finding**: v2 said the monolith risk (bad blog deploy breaks login/billing; blog-agent
cron competes for Lambda concurrency with interactive traffic) was "named... with a stated
mitigation path (Lambda aliases + versioning, reserved concurrency)" — but neither mitigation was
ever scheduled in either document's execution plan. That's the same failure mode §4.7's Lambda-
warmer fix explicitly rejected elsewhere in this same document pair (a named-but-uninstrumented
risk). Corrected: **these mitigations are not scheduled in this revision** — they remain accepted,
unmonitored risks at current near-zero traffic. If either risk becomes real (a blog deploy actually
breaks login, or the blog-agent cron visibly delays interactive requests), that's the trigger to
schedule Lambda aliasing/versioning as a dedicated follow-up — not something to claim is already
mitigated. Decision unchanged (monolith stays); honesty about what's actually protected against it
is the fix.

### 4.3 Blog write-endpoint hardening — round-2 fix: the proposed mechanism was technically wrong

**Round-2 finding, confirmed live**: v2 proposed "an API Gateway usage plan with a low per-route
throttle" for craighoad-blog's admin-write endpoint. Checked directly: `fhu7jtpt1l` is an API
Gateway **v2 HTTP API**; Usage Plans (and the API-key-tied throttling they imply) are a **REST
API (v1)-only construct** — zero usage plans exist in this account, and none can be attached to
an HTTP API. Following the v2 instruction as written would send an implementer down a dead end.
**Corrected mechanism**: `route_settings` throttle limits
(`throttling_burst_limit`/`throttling_rate_limit`) on the `aws_apigatewayv2_stage` resource,
scoped to the write route specifically — achieves the same near-$0 rate-limiting goal via the
Terraform resource this API type actually supports. Full checklist now lives in the roadmap doc's
Stage 5 (the properly-numbered version of what v2 called "Stage 1b" — see §9).

### 4.4 Tag-enforcement guardrail — round-2 fix: actual condition list, real test method, both accounts covered

**Round-2 findings**: v2 committed to SCP over Config (that part held up) but never specified the
actual condition structure, and "dry-run/simulate" cited a testing method that doesn't exist for
SCPs (no native AWS SCP-simulate API). Also, v2 scoped this guardrail to `hcp-terrorgems-prod`
only, but the properly-integrated Stage 5 (craighoad-blog wiring, §4.3) touches
`hcp-craighoad-prod` — a different account with no equivalent protection. Both fixed in the
roadmap doc's Stage 3: a concrete `Deny`-with-`RequestTag`-condition SCP, instantiated per-account
via Terraform module with each account's own allowed `Project` value hardcoded, tested for real by
attaching to `hcp-qa` first (not a hypothetical "simulate"), then rolled out to **both**
`hcp-terrorgems-prod` and `hcp-craighoad-prod`. The doc also now states the guardrail's real
limits honestly: it only covers an explicit action list and only where the `RequestTag` condition
key is supported at creation time — not a complete guarantee.

---

## 5. Migration Plan — now genuinely one combined sequence, not two independently-numbered lists with a gap between them

**Round-2 finding, the most structurally important one from that review**: v2 claimed this section
was "a single combined execution order shared with the companion roadmap document," but the
roadmap's own numbered stages never actually included "Stage 1b" (this document's craighoad-blog
work) — it existed only as prose in this document, unguarded by the tag-enforcement guardrail
(different account), with no checklist, acceptance criteria, or rollback note, unlike every stage
in the roadmap's real sequence. **Fixed in v3**: the roadmap document's §6 now has a real,
numbered Stage 5 for this work, sequenced after Stage 3 (the guardrail, now extended to cover
`hcp-craighoad-prod` too). This document no longer maintains its own parallel stage list — the
roadmap doc's §6 is the single authoritative sequence for both documents, and this section just
points to it:

- **Stage 0** (roadmap doc): secrets hardening — now includes the required code change, not just
  config (round-2 fix, see roadmap §4.6).
- **Stage 1** (roadmap doc): mechanical cleanup — now includes the second orphaned KMS key, real
  CLI methodology for PITR/CloudTrail-lookback checks, and an honest (not unfalsifiable) acceptance
  criterion.
- **Stage 2** (roadmap doc, new in v3): the SSM-contamination-origin investigation, split out of
  Stage 1 and properly timeboxed — it was an unbounded item buried in a mechanical checklist in v2.
- **Stage 3** (roadmap doc): tag-enforcement guardrail, now covering both accounts.
- **Stage 4** (roadmap doc): formalize `hcp-qa`.
- **Stage 5** (roadmap doc, new in v3 — this is v2's orphaned "Stage 1b," now real): craighoad-blog
  audit and hardening, with the corrected throttle mechanism from §4.3 above.
- **Stages 6–8b** (roadmap doc): TMDB pipeline, auth hardening, bounded fixes, gamification audit.
- **From this document specifically — status as of 2026-08-12**: archive the dead repos — ❌ not
  started, and now only 2 candidates, not 3 (§3.1–3.6 correction above); extract the shared
  static-hosting pattern into a reusable Terraform module — ❌ not started, no `static-hosting`
  module exists anywhere in `aws-terraform-platform-aws-modules`; apply consistent security
  headers — 🟡 partial, terrorgems.com has a full response-headers policy live, craighoad.com has
  none; add cost-allocation tags to separate craighoad.com from ai.craighoad.com — 🟡 partial, the
  tags physically exist on both distributions' resources but are **not activated** in Billing
  Preferences (`aws ce list-cost-allocation-tags` shows them `Inactive`) — Cost Explorer still
  cannot split the totals until someone runs
  `aws ce update-cost-allocation-tags-status --cost-allocation-tags-status
  TagKey=Project,Status=Active TagKey=repo,Status=Active` (or the equivalent Billing console
  toggle) — this is a real, cheap, unblocked next step that was missed.

**Total effort**: see the roadmap doc's §6 for the authoritative total (~10–13 days + two
timeboxed half-days + one unscoped audit, up from v2's 9–11 once Stage 2's split and Stage 5's
real scope are counted honestly instead of estimated as a parenthetical).

---

## 6. Open Items Requiring a Decision or One More Check

- **GCP check**: still unresolved — requires an interactive `gcloud auth login` that couldn't be
  completed non-interactively in any session so far.
- **ai.craighoad.com cost separation**: tags now exist on both distributions (§5), but are not
  yet **activated** in Billing Preferences — that's the actual remaining step, not adding more
  tags.
- **SSM contamination origin**: now the roadmap doc's Stage 2, properly timeboxed (half a day),
  rather than an open-ended item — see that document.
- **DNS quarantine runbook**: not written, referenced in §2.3's honest tradeoff statement as the
  thing that would make that tradeoff actually quantifiable if it existed. Out of scope for this
  revision; noted for whoever picks it up.

---

## 7. Corrections to Prior Documentation

Earlier planning artifacts described a different account structure ("rhyscraig org") than what's
live (`hcp` org) — corrected in v2. The pattern that repeated across three review rounds since:
verify the claim, then verify the fix, then verify the fix of the fix. Don't assume a revision is
correct because it says "corrected" — that's why §8 exists.

## 8. Review History (condensed — full per-round detail lives in the roadmap document's §8)

Three adversarial rounds, five independent reviewers each, live AWS/repo access. Round 1 found the
false blog-deployment claim and an incomplete contamination inventory. Round 2 found the orphaned
"Stage 1b" (a stage this document introduced but never actually joined the roadmap's numbered
sequence), the wrong API Gateway mechanism, and an unquantified DNS tradeoff dressed as quantified.
Round 3 found the WAF and tax figures round 2 "corrected" were themselves still wrong (re-verified
against real invoices), and — the finding that lives primarily in the roadmap document since it's
Terraform-config-specific — a real IAM logic bug in the tag-enforcement SCP design that would have
left `hcp-craighoad-prod` unprotected despite the guardrail claiming to cover it.

This document's specific fixes across the three rounds: craighoad-blog's deployment status
corrected (§2.1); the tag-enforcement guardrail's mechanism corrected, then its logic bug fixed —
see roadmap §8/Stage 3 for the full detail (§4.4); the DNS tradeoff reframed as an honest
unquantified judgment call (§2.3); the cost model corrected twice for missing categories (KMS,
Secrets Manager, Tax/VAT) and once more for wrong unit prices (§2.4); "Stage 1b" replaced by the
roadmap's real, numbered Stage 5 (§5). If this document changes substantively again, repeat the
adversarial pass rather than trusting self-review.

## 9. Execution Progress (new in v5 — 2026-08-12)

A real engineering session executed against v4, in parallel with a separate session working the
TerrorGems-side roadmap stages. This section is ground truth from that execution, not another
review round — the difference matters: a review round finds problems with a *design*; this found
problems with *reality* that the design (even a well-reviewed one) couldn't have anticipated.

**What's actually done**: Stage 3 (tag-enforcement guardrail) is largely built — the SCP exists,
is attached to both target accounts and `hcp-qa`, uses correct per-account values, and the
executing session independently found and fixed a real logic bug in the round-3 design (a single
ANDed Deny statement doesn't catch a present-but-wrong-value tag any better than it catches an
absent one — fixed by splitting into two separate Deny statements, which is a better design than
this document ever specified). Stage 5's bounded audit ran and confirmed craighoad-blog is not
wired into CloudFront — but that finding was never written back into either document until now.

**What's blocking, found only by executing, not by reviewing**:
- A tag-casing bug (`Project` vs `project`) means the guardrail currently wouldn't protect the
  account it's meant to — see roadmap doc Stage 3 for the fix.
- craighoad-blog has a live 404 bug (API Gateway stage-prefix mismatch) that's made it
  non-functional for all real traffic since it was deployed — completely independent of anything
  either plan anticipated, and it blocks Stage 5's core acceptance criterion regardless of wiring
  or hardening status.
- The blog's write endpoint has no authentication at all — worse than Stage 5's original
  assumption that a secret needed "migrating."
- One of the three repos this document classified as dead (§3.1–3.6) is not dead — corrected above.
- This document itself had never been committed to git before this revision — fixed now.

**Lesson for whoever picks this up next**: a design that survives three adversarial review rounds
can still be wrong about facts that don't exist until execution creates them (a no-op apply that
never really tested anything, a tag applied with the wrong case, a bug that's been live the whole
time but nobody hit the real endpoint to find it). Review catches design flaws; only running the
thing catches this class of problem. Keep both disciplines — don't let three clean review rounds
create false confidence that execution won't surface anything new.
