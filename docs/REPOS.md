# Repo Inventory

Every repo touching AWS infrastructure, its purpose, its current CI/CD model, and its migration
status toward the target design in [SECRETS.md](SECRETS.md), [CICD_ROLES.md](CICD_ROLES.md), and
[BACKEND.md](BACKEND.md). Findings below come from direct inspection (workflow YAML, backend
config, `gh run list` CI history, and full-repository BeyondTrust-content greps) during the audit
that produced this documentation — not from assumption. Both `rhyscraig` and `hoad-org` GitHub
orgs are covered — **`rhyscraig` was renamed/repos progressively transferred to `hoad-org`**, but
this migration is incomplete and inconsistent: some repos still live under `rhyscraig` with no
`hoad-org` counterpart, some redirect, some have been transferred outright. Treat each repo
individually — don't assume a blanket migration.

## The central architectural fact

**`hoad-org/aws-terraform-platform-aws-workflows` is a separate, older, less mature
reusable-workflow repo than `hoad-org/github-automation`** — not an earlier name for the same
thing. Different design entirely: GitHub Environment secrets (not Secrets Manager), no
`get-aws-secrets`-style OIDC-derivation, hardcoded/duplicated OPA policies, broken self-references
(checks out its own stale `rhyscraig/` name; `pip install`s from `rhyscraig/tfctl.git` and
`rhyscraig/awsctl.git`, neither of which exist under those names — the real repos are
`aws-python-platform-tfctl` and `awsctl-platform-aws`).

**Every one of the following repos currently consumes the older `aws-workflows` repo, either via
its `rhyscraig/` alias or the `hoad-org/` name directly**: `aws-accounts`, `aws-baselines`,
`aws-org`, `aws-terraform-platform-aws-templates`, `aws-terraform-platform-aws-modules`,
`aws-terraform-solutions-terrorgems-platform`,
`personal-ai-cloud`. **Zero repos consume `github-automation` yet** except the seed repo's own new
`deploy.yaml` (this session). **Migrating every caller from `aws-workflows` to `github-automation`
is the actual redesign work — not a green-field build.**

`aws-terraform-solutions-websites` (previously listed here) has been **retired/archived**
(2026-07-16) rather than migrated — see "Retired repos" below.

## Summary table

| Repo | Org | Naming OK | Secrets model | State backend (real) | CI health | BT content |
|---|---|---|---|---|---|---|
| `aws-terraform-platform-seed` | hoad-org | ✅ | `github-automation` | `hcp/aws/control-plane/terraform.tfstate` | green — `deploy.yaml` verified end-to-end (parse-config, plan, approve, apply) | **Yes** — `configs/orgs/{bt-avm,bt-dev,fdr-cmc,fdr-gvc}.tfvars` (deleted), README/AI.md still reference BT clone URLs (not yet fixed) |
| `aws-terraform-platform-docs` | hoad-org | n/a | n/a | n/a | n/a | this repo |
| `aws-terraform-platform-aws-accounts` | hoad-org | ✅ | old `aws-workflows` + `AWS_CI_ROLE_ARN`/`GH_PAT` | `hcp/prd/platform/aws-accounts/eu-west-1/terraform.tfstate` | drift-check red (3+ failures) | **git history only** — `aws-avm-{audits,security,shared-svcs}@beyondtrust.com`, author `choadbt <choad@beyondtrust.com>` |
| `aws-terraform-platform-aws-baselines` | rhyscraig | ✅ | old `aws-workflows` + `AWS_CI_ROLE_ARN`/`GH_PAT` | `hcp/prd/platform/aws-baselines/eu-west-1/terraform.tfstate` (second dead backend block confirmed and removed — PR #5, ADR-0001) | drift_check red (5/5 failures) | none in current tree |
| `aws-terraform-platform-aws-org` | hoad-org | ✅ | old `aws-workflows` + `AWS_OIDC_ROLE_ARN`/`GH_PAT` | `hcp/prd/platform/aws-org/eu-west-1/terraform.tfstate` | drift-check red (5/5 failures, every 4h) | **git history only** — author `choadbt <choad@beyondtrust.com>` |
| `aws-terraform-platform-aws-templates` | rhyscraig | ✅ | old `aws-workflows` + `SEED_ROLE_ARN`/`TF_STATE_BUCKET`/etc | via `-backend-config`; `.tfctl.yml` cites a **different management account** (`235494790978`) than everywhere else (`395101865577`) — unresolved discrepancy | no real deploy runs in recent history | ambiguous — `.tfctl.yml` bucket pattern `bt-terraform-remote-state-{region}` |
| `aws-terraform-platform-aws-modules` | rhyscraig | ✅ | old `aws-workflows`, no deploy (pure module catalog) | none (no backend anywhere — correct, it's a module registry) | green (Dependabot only) | none |
| `aws-terraform-platform-aws-workflows` | hoad-org | ✅ (but see "central architectural fact" above) | n/a — this **is** the older secrets/workflow provider | n/a | n/a | none directly, but see "Adjacent tooling" below |
| `aws-terraform-solutions-terrorgems-platform` (renamed from `-terrorgem`) | hoad-org | ✅ | bespoke `terraform-terrorgem-prd.yml`, not on `github-automation` — `SEED_ROLE_ARN`/`PLAN_PASSPHRASE`/`PROD_CICD_ROLE_ARN`/`TF_LOGS_BUCKET`/`TF_STATE_BUCKET`/`KMS_KEY_ID` live as **environment secrets on `terrorgem-prd`**, not repo secrets — easy to miss with `gh secret list` (repo-scope only); `JWT_SECRET`/`TMDB_API_KEY` (app secrets) are repo-scope | root `backend.tf` orphaned/stale; real backends per-environment under `infra/aws/environments/*/backend.tf`; `scripts/deploy.sh` hardcodes a bucket/region that **doesn't match** the CI workflow's region | **`ci.yml` on `main` red since 2026-04-26** (Python ruff/mypy/pytest/contracts/governance AND Frontend typecheck/lint/vitest both failing, confirmed live 2026-07-29, run 29491753794) — app-level, not infra. `terraform-terrorgem-prd.yml` only triggers via `workflow_run` after `ci` succeeds on `main`, so it has been silently **skipped** on every push since, not failed — 52 real successes before that, most recently 2026-04-24, so the deploy pipeline itself isn't known-broken, it's just never been reachable. Not investigated further this pass: app-level CI failures need product-code context this audit doesn't have. `PROD_CICD_ROLE_ARN`/`SEED_ROLE_ARN` also not verified live for staleness — untested since April. | none |
| `aws-terraform-solutions-craighoad-blog` | hoad-org (redirects from rhyscraig) | ✅ | inline `SEED_ROLE_ARN` only, no reusable workflow calls at all | `terraform/backend.tf` (hardcoded) | **never actually executed** except Dependabot | none |
| `craighoad-portfolio-website` (renamed from `website-static-html-craighoad.com`) | hoad-org | ❌ not `aws-terraform-*` prefixed | unknown — not independently re-audited this pass beyond OIDC trust | unknown | unknown | not re-confirmed |
| `personal-ai-cloud` | hoad-org | ❌ not `aws-terraform-*` prefixed (defensible exception — app repo with embedded infra) | old `aws-workflows` (`hoad-org/` direct, not the `rhyscraig/` alias) — **not yet migrated to `github-automation` despite that being built this session for it** | `hcp/prd/personal-ai-cloud/*` (target: `workloads/craighoad.com/personal-ai-cloud/`) | 5/5 failures (all within 24h — likely the now-fixed OIDC subject issue, unconfirmed re-run) | none |
| `github-automation` | hoad-org | n/a — shared workflow hub, correctly unprefixed | is the target model — AWS Secrets Manager / Azure Key Vault, zero GH secrets | n/a | n/a (pure `workflow_call` library) | none |

## Adjacent tooling with real, unresolved BeyondTrust content — flag prominently

These aren't `aws-terraform-*` repos, but they're actively used to operate this landing zone
(including by Claude, this session, via `cloudctl`) and still carry real BeyondTrust content:

- **`hoad-org/cloudctl`** — the credential-vending tool used throughout this session to get real
  AWS access. Its own README says **"internal use by BeyondTrust Engineering only"**, its origin
  remote references `BT-IT-Infrastructure-CloudOps/aws-terraform-infra-cloudops-cloudctl`, and its
  own usage examples use `bt-avm` as the example org name throughout. This needs a real decision:
  is this Craig's own tool (in which case it needs a full rebrand/decouple from BT), or is it
  something that should stop being used for personal infrastructure entirely?
- **`hoad-org/awsctl-platform-aws`** — same pattern: README says "internal use by BeyondTrust
  Engineering only," installs via `pipx install "git+https://github.com/BT-IT-Infrastructure-CloudOps/awsctl.git@v2.8.1"`.
- **`rhyscraig/aws-python-platform-tfctl`** — README says it "Fetches secrets (AWS Secrets
  Manager, BeyondTrust)" and documents installing from
  `BT-IT-Infrastructure-CloudOps/tfctl.git`. Referenced (and broken — wrong repo name) from CI
  workflows in `aws-org` and the old `aws-workflows` repo.

**None of this has been assessed for whether it's safe to keep using** — flagging here rather than
making a unilateral call, since "is it OK to keep running a tool whose own README claims it's for
BeyondTrust-internal use only" is a real, non-technical judgment call.

## Stale OIDC trust subjects — fixed

The seed repo's `github_oidc_subjects` list (the legacy combined-role trust list) had two entries
referencing renamed/moved repos that would never match a real token again — removed:
- `repo:rhyscraig/aws-terraform-solutions-terrorgem:*` — the real repo is
  `aws-terraform-solutions-terrorgems-platform` (renamed).
- `repo:rhyscraig/website-static-html-craighoad.com:*` — the real repo is
  `hoad-org/craighoad-portfolio-website` (renamed **and** transferred org, already trusted
  separately under its real name).

These never granted excess access (a stale subject just never matches) — flagged here only so a
future session doesn't waste time re-discovering that those old names are
still meaningful.

## Retired repos

- **`aws-terraform-solutions-websites`** — archived 2026-07-16, not migrated to
  `github-automation`. While auditing its CI/CD migration, found that it defines an S3 bucket +
  CloudFront distribution + Route53 records for `www.craighoad.com`/`craighoad.com`, targeting AWS
  account `767828739298` (terrorgems) via `terraform.tfvars`' `prod_account_id`. That account has
  no matching state bucket and never had these resources — the repo's Terraform had in fact never
  been successfully applied there (CI had been failing since 2026-03-28, most recently on a stale
  GitHub billing lockout from before the `hoad-org` transfer, unrelated to the account-ID bug).
  Live-queried the real `craighoad.com` infrastructure instead: the actual, currently-serving S3
  bucket + CloudFront distribution + Route53 zone live in account `624426145233` (craighoad-prod),
  tagged `Repo: website-static-html-craighoad.com` (the old name for what's now
  `hoad-org/craighoad-portfolio-website` — a **separate repo** that already owns and manages this
  domain). Fixing this repo's account ID and applying it for real would have attempted to create
  resources with the same names as ones already live under a different repo's Terraform state — a
  real collision risk to the live site. The repo's other configured domain,
  `cobwebsandcorridors.com`, has no hosted zone anywhere in the org and doesn't resolve — dormant/
  unclaimed, not worth salvaging on its own. Retired rather than partially fixed.

## Naming convention violations

- `craighoad-portfolio-website` — not `aws-terraform-*` prefixed, despite provisioning real AWS
  infrastructure. Either rename or explicitly document as an accepted exception.
- `personal-ai-cloud` — same issue, more defensible (app repo with embedded infra under
  `infra/aws/`) — should still be an explicit, documented exception.
- `aws-terraform-platform-docs` (this repo) and `aws-terraform-platform-seed` are correctly
  prefixed but are non-standard in their own way: `docs` isn't Terraform/AWS infra itself (by
  design), and `seed` had no working CI/CD pipeline at all until this session.

## Currently-broken CI, ranked by urgency

**Update, 2026-07-29 — all five items below are resolved.** `aws-org` migrated onto
`github-automation` and its dead `drift-check.yaml` cron deleted (root cause: every job passed a
GitHub secret, `AWS_OIDC_ROLE_ARN`, that never existed on the repo — confirmed
`aws-terraform-platform-aws-org` PR #10). `aws-baselines` and `aws-accounts` were already migrated
by the time of this pass; re-run and confirmed green. `personal-ai-cloud` and
`aws-terraform-solutions-craighoad-blog` re-run and confirmed green (7/7 and 8/8 jobs). Left here
for history, not as a current list — see each repo's own Actions tab for live status rather than
trusting this section.

1. ~~`aws-org` — every 4-hour drift-check cron failing, 100% failure rate.~~ Fixed.
2. ~~`aws-baselines` — daily drift-check failing 5/5 consecutive days.~~ Fixed.
3. ~~`aws-accounts` — drift-check failing (3/5 recent).~~ Fixed.
4. ~~`personal-ai-cloud` — deploy failing 5/5 in the last 24h.~~ Fixed.
5. ~~`aws-terraform-solutions-craighoad-blog` — had never executed outside Dependabot.~~ Fixed.

**New entry, 2026-07-29**: **`aws-terraform-solutions-terrorgems-platform`** — `ci.yml` on `main`
has been red since 2026-04-26 (Python ruff/mypy/pytest/contracts/governance AND Frontend
typecheck/lint/vitest both failing). This is app-level, not infra — see the Summary table row
above for the full finding, including that the deploy pipeline itself has 52 real historical
successes and isn't known-broken, it's just been unreachable (its trigger requires `ci` green on
`main`). Needs product-code investigation this audit doesn't have context for; not attempted this
pass.

## Cross-repo backend/region inconsistencies (see [BACKEND.md](BACKEND.md) for the canonical target)

At least **four distinct state-bucket-naming conventions** coexist: `hcp-cmc-euw1-platform-tfstate-prd`
(the real, live one, used consistently by `aws-accounts`/`aws-baselines`/`aws-org`),
`infra-tfstate-hoad-org-seed` (referenced in `environments.yml`/`.tfctl.yml` in those same repos,
but not matching their actual `backend.tf` — looks like dead/legacy config, not confirmed dead),
`bt-terraform-remote-state-{region}` (`aws-terraform-platform-aws-templates`'s `.tfctl.yml`), and
`prd-<account>-eu-west-2-tf-state-bucket` (`aws-terraform-solutions-websites`'s `backend.hcl` — that
repo is now retired, see "Retired repos" above; this bucket never actually existed).
Region values disagree within single repos too (`eu-west-1` vs `eu-west-2` vs `us-east-1` cited in
different config files for the same environment, in the same repo, in multiple repos).

## Per-repo migration checklist (for any repo not yet on the target model)

Every item below is something that actually broke a real deploy.yaml run while building the seed
repo's own pipeline — this isn't a theoretical checklist, each line is load-bearing.

- [ ] State key matches [BACKEND.md](BACKEND.md)'s convention, declared in a config file, not
      hardcoded in workflow YAML (the seed repo itself is the one deliberate exception — see
      BACKEND.md)
- [ ] `deploy.yaml`'s first job is `plan_parse-config` (`reusable-tf-parse-config.yaml`) — every
      other job reads `needs.plan_parse-config.outputs.*` for `aws_region`/`aws_account_id`/
      `partition`/`apply_env`, none of them hardcoded as literals. Add `pr-validate.yml` alongside
      it (same gates, no approve/apply/audit) — see `aws-terraform-platform-seed`'s copy of both
      files for the exact pattern to replicate.
- [ ] Workflow calls `hoad-org/github-automation`'s reusable workflows and `get-aws-secrets`, not
      `aws-terraform-platform-aws-workflows` (either org) or raw GitHub secrets
- [ ] `checkov-scan`/`quality-gate`/`plan-encrypt`/`apply-decrypt` all pass
      `require_github_token: "false"` **unless the repo genuinely has private git-sourced
      Terraform modules** — omitting this means every one of those jobs fails on the
      never-created `github/github-automation/pat-token` secret. `approve-gate`/`audit` never need
      it at all (hardcoded false inside those reusable workflows, not a caller-facing input).
- [ ] No `secrets.*` reference to `SEED_ROLE_ARN`, `AWS_CI_ROLE_ARN`, `AWS_OIDC_ROLE_ARN`,
      `TF_STATE_BUCKET`, `KMS_KEY_ID`, or `PLAN_PASSPHRASE` anywhere in the repo's workflows — and
      the repo's GitHub secrets (`gh secret list` / `gh api .../environments/{env}/secrets`) are
      actually deleted after cutover, not just unused
- [ ] OIDC subjects follow the exact formula in [CICD_ROLES.md](CICD_ROLES.md)'s "real OIDC
      subject list" section — **derived empirically, don't guess by job name**:
      `github_oidc_plan_subjects` needs `ref:refs/heads/main` + `environment:{org}` +
      `environment:{org}-approve` (three distinct callers of `get-aws-secrets`, which always
      authenticates as the plan role); `github_oidc_apply_subjects` needs `ref:refs/heads/main`
      (apply-decrypt's real apply step sets no `environment:`), plus
      `environment:{org}-approve` too if the repo has its own extra post-apply job that sets an
      environment (e.g. `personal-ai-cloud`'s `deploy_portal_files`). Use the repo's **current**
      name/org, not a stale pre-rename one.
- [ ] The `{org}-approve` GitHub Environment exists (`gh api repos/{owner}/{repo}/environments/
      {org}-approve -X PUT`) — and don't expect it to enforce a real pause: required-reviewers and
      wait-timer protection rules both need a paid GitHub plan on a private repo (confirmed live
      this session). `deploy.yaml` should be `workflow_dispatch`-only, no `push: branches: [main]`
      trigger — that's the $0-tier substitute gate.
- [ ] `expected_account_id`/org/OU IDs (if any) verified against a live AWS API call, not copied
      from another repo, a template, or an SSO URL
- [ ] Zero BeyondTrust content (grep for `beyondtrust`, `BT-IT-Infrastructure`, `bt-avm`, `bt-dev`,
      `btsb`, `btc`, `@beyondtrust.com` across every file — and check git history, not just the
      current tree, if the repo has ever committed real credentials/emails)
- [ ] No dependency on `aws-python-platform-tfctl`/`awsctl-platform-aws`/`cloudctl` until their own
      BeyondTrust-content status is resolved (see "Adjacent tooling" above)
- [ ] Verify locally before pushing, don't round-trip through CI to iterate: `terraform fmt
      -check`/`validate`, and for Checkov specifically, run the exact CI invocation locally
      (`checkov -d <dir> --framework terraform --compact --quiet --soft-fail
      --download-external-modules true --hard-fail-on "CKV_AWS_24,CKV_AWS_25,CKV_AWS_277,
      CKV_AWS_20,CKV_AWS_57,CKV2_AWS_6,CKV_AWS_17"`) before pushing a fix and burning Actions
      minutes on a guess.
- [ ] `hoad-org/aws-terraform-platform-docs` updated in the same change (this repo — see the
      guardrail in [PROVENANCE.md](PROVENANCE.md))
