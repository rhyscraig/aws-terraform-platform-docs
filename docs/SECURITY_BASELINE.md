# Security Baseline — maximum hygiene achievable at $0

This is the security posture this landing zone actually runs, given the [$0 budget](BUDGET.md).
Everything below is free. Nothing below is a compromise for cost reasons — it's genuinely the
right baseline regardless of budget. What's *not* here (GuardDuty, Config, Security Hub, Macie) is
absent because those cost money, not because they were overlooked — see the end of this file and
[SECURITY_ROADMAP.md](SECURITY_ROADMAP.md).

> **Correction, 2026-07-29** — a live sweep of all 7 accounts across
> eu-west-1/eu-west-2/us-east-1 found that four things this file described as active controls did
> not exist anywhere: CloudTrail, IAM Access Analyzer, account-level S3 block-public-access, and
> enforced PR review. This file asserted a posture that was not real.
>
> CloudTrail and Access Analyzer have since been **built and verified live** (see below). The
> other two remain open. **Treat every line in this file as a claim to verify, not a given, until
> it has a live check behind it** — the failure mode here wasn't a wrong decision, it was
> documentation drifting ahead of implementation and then being trusted.

## On, by design

- **CloudTrail (management events)** — org-wide, always on. **This was documented here for months
  before it existed**: on 2026-07-29 `aws cloudtrail describe-trails` returned empty in every
  account and every approved region (the only trail anywhere was a legacy `asatst-prod-main-trail`
  in hcp-terrorgems-prod), meaning `ProtectAuditLogs` (SCP `p-leebx0ta`) guarded nothing and there
  was no durable audit record beyond Event History's rolling 90 days. **Built and verified live
  the same day** — `hcp-org-trail`, organization trail, multi-region, global service events, log
  file validation on, `IsLogging: True`, delivering to `hcp-cmc-euw1-platform-cloudtrail-prd` in
  hcp-log-archive (which until then was a vended account with no purpose at all). Confirmed
  receiving real objects under `AWSLogs/o-h07s8pk406/<account>/` from multiple member accounts
  across multiple regions. Because it's an organization trail created in the management account,
  every future AVM-vended account is covered automatically with no per-account Terraform. Data
  events (S3 object-level, Lambda invocations) are NOT enabled — `get-event-selectors` returns no
  data resources, and that emptiness *is* the cost control: management events are free, data
  events bill per event. Managed in `aws-terraform-platform-aws-org/cloudtrail.tf`.
- **IAM Access Analyzer** — free, flags unintended external/cross-account access. **Same story**:
  documented here but `aws accessanalyzer list-analyzers` returned empty in all 7 accounts and all
  three regions on 2026-07-29. Now `hcp-org-external-access`, **ORGANIZATION** scope so one
  analyzer covers every member account — account scope would have seen none of the member-account
  side of this estate's real external-access surface (the two-hop OIDC chain, the StackSet-vended
  member roles, the shared state bucket's cross-account grants). Managed in
  `aws-terraform-platform-aws-org/security.tf`. Findings are not routed anywhere yet — no
  EventBridge rule, no SNS — because there's no habit of triaging them yet; wire that up once
  there is.
- ~~**S3 block-public-access** — on for every bucket, always, no exceptions.~~ See the corrected
  entry below — the account-level control is absent and the SCP that was meant to enforce it was
  never deployed.
- **MFA / root credentials** — management account root has MFA enabled. All six **member** accounts
  had `AccountMFAEnabled=0`, and hcp-terrorgems-prod had a live root password with no MFA on it.
  Resolved 2026-07-29 by enabling AWS Organizations centralized root access
  (`enable-organizations-root-credentials-management` + `enable-organizations-root-sessions`) and
  deleting root credentials from all six: every member account now has **no root password, no root
  access keys, no root MFA device**, verified via `IAMAuditRootUserCredentials` sessions. Root is
  reachable only via `sts:AssumeRoot` from the management account. Note this required fixing
  `DenyRootUser` (SCP `p-7a6dyw58`) first — as written it denied root `Action: ["*"]`, which
  blocked root from deleting its own credentials, making the exposure permanent. See
  `hoad-org/aws-terraform-platform-aws-org` PR #8.
- **Least-privilege IAM** — the split plan/apply/module-skills OIDC role model (see
  [CICD_ROLES.md](CICD_ROLES.md)) exists specifically for this: a compromised plan-phase token
  (runs unattended, on every push) can never mutate real infrastructure, full stop, not just by
  convention.
- **S3 block-public-access** — set per-bucket by the seed repo's own Terraform (state + logs
  buckets pass all four `block_*`/`ignore_*`/`restrict_*` flags). **The account-level control is
  NOT set**: `aws s3control get-public-access-block` returns `NoSuchPublicAccessBlockConfiguration`
  in every one of the 7 accounts, so nothing stops a *new* bucket being made public. The SCP meant
  to backstop this (`DenyS3PublicAccess`) was written in
  `aws-terraform-platform-aws-baselines/terraform/security/main.tf` but **never deployed** — that
  module isn't called, and wouldn't apply if it were (it references a deleted
  `aws_kms_key.cloudtrail`, writes lifecycle rules to the non-existent bucket
  `infra-tfstate-hoad-org-seed`, and attaches an SCP to an org ID rather than a root ID). A live
  `list-policies` shows 6 SCPs; that isn't one of them. `aws-terraform-platform-aws-modules`'
  `security-baseline` module has the correct free resources
  (`aws_s3_account_public_access_block`, `aws_ebs_encryption_by_default`,
  `aws_iam_account_password_policy`) and **has zero consumers**.
- **KMS encryption at rest** — the shared state bucket and (where applicable) workload data stores
  use KMS, not plaintext.
- **Native S3 state locking** (`use_lockfile = true`) — prevents concurrent-apply corruption,
  free (replaces the older DynamoDB-lock-table pattern, which would cost money at any real scale).
- **AWS Organizations SCPs** — free, the primary technical enforcement layer for the cost
  guardrail in [BUDGET.md](BUDGET.md), and available for genuine security guardrails too (e.g. deny
  root-user API calls, deny leaving the org, deny disabling CloudTrail) — worth expanding over
  time, at zero cost.
- **`main`-only branch discipline, PR-required for every change** — see [PROCESS.md](PROCESS.md).
  **Convention only, not enforced.** Confirmed live 2026-07-29: every private repo returns
  `"Upgrade to GitHub Pro or make this repository public to enable this feature"` (HTTP 403) for
  both `/branches/main/protection` and `/rulesets`. Branch protection exists on exactly one repo —
  the public `aws-terraform-platform-aws-modules`. Anyone or anything with write access can push
  straight to `main` on every private repo in the estate, and `main` is what the apply-role OIDC
  trust policy keys on. Making the platform repos public is the $0 fix (it also unlocks secret
  scanning, push protection and unmetered Actions minutes); paying for GitHub Pro/Team is the
  other. Until one of those happens, do not describe this as a control.
- **GitHub org 2FA** — `two_factor_requirement_enabled: false` on `hoad-org` as of 2026-07-29. The
  GitHub org is the real control plane for this AWS estate (every mutation path starts with a
  `repo:hoad-org/...:ref:refs/heads/main` OIDC claim), so this is the highest-leverage free control
  available and it is currently off. The org's only member already has 2FA on their personal
  account, so enforcement can be switched on without locking anyone out.

## Deliberately off — cost, not oversight

- **GuardDuty** — 30-day free trial, then a real, usage-based cost. Off.
- **AWS Config** — no free tier at all; cost per configuration item recorded + per rule
  evaluation. Off.
- **Security Hub** — aggregates findings from GuardDuty/Config/Inspector, all of which cost money
  themselves. Off (nothing to aggregate yet anyway).
- **Macie** — per-object/per-GB scanning cost. Off.
- **VPC Flow Logs to CloudWatch Logs** — CloudWatch Logs ingestion/storage costs at volume. Off
  unless actively debugging (turn off again after).
- **X-Ray** — free tier is generous but not infinite; not enabled by default, opt in per-repo if a
  specific debugging need justifies it.

Every one of these is denied at the IAM layer for the Workloads OU by the SCP in
[BUDGET.md](BUDGET.md) — specifically so a future Claude session can't "helpfully" turn one on
without it being a deliberate, reviewed decision.

## What a real incident response looks like at $0

CloudTrail management events + IAM Access Analyzer + the OIDC trust-subject model (every action is
attributable to a specific repo/workflow run, not a shared long-lived credential) already give
enough signal to reconstruct "what happened and from where" for anything short of a
sophisticated, sustained intrusion. That's the realistic bar at this budget and this threat model
(personal infrastructure, not a company handling other people's regulated data) — see
[SECURITY_ROADMAP.md](SECURITY_ROADMAP.md) for what changes once that's no longer true.
