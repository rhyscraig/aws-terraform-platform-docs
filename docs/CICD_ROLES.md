# CI/CD OIDC Roles

## Status, 2026-07-29 — the combined role is down to one repo

The legacy combined `workloads_oidc_role` (`hcp-cmc-euw1-platform-oidc-role`) attaches
`seed_oidc_policy` — `iam:*`, `sts:*`, `s3:*`, `kms:*`, `cloudformation:*`, `organizations:*` —
which is **org-admin in the management account**. Until 2026-07-29 it was trusted by **six** repos:
the seed, `aws-org`, `terrorgems-platform`, `craighoad-portfolio-website`, `personal-ai-cloud` and
`craighoad-blog`. Compromise of any one of them was org takeover.

It is now trusted by exactly one — the seed itself, which genuinely needs those permissions
because it manages the OIDC provider, all three roles, the KMS key, both S3 buckets and the
StackSet. Live trust policy:

```
repo:hoad-org/aws-terraform-platform-seed:ref:refs/heads/main
repo:hoad-org/aws-terraform-platform-seed:environment:hcp
repo:hoad-org/aws-terraform-platform-seed:environment:hcp-approve
```

Two things worth carrying forward from how that was done:

- **Four of the five removals turned out to be excess privilege that was never used at all.**
  `craighoad-blog` and `personal-ai-cloud` authenticate against craighoad-prod's *own* plan/apply
  roles (confirmed by reading 624426145233's live trust policies); `craighoad-portfolio-website`
  was already on the split lists; `terrorgems-platform`'s pipeline references three GitHub secrets
  that don't exist on it and has never completed. Verify what a grant is actually used for before
  assuming a removal is risky — here, most of the risk was in keeping it.
- **The role is kept, not deleted.** The KMS key policy, the state bucket policy and the
  member-role StackSet trust all still name it by ARN.

**Still open — the plan role is not actually read-only.** `plan_oidc_policy` grants
`sts:AssumeRole` on `arn:aws:iam::*:role/hcp-cmc-euw1-platform-cicd-role`, and that StackSet-vended
member role carries `iam:*`, `s3:*`, `kms:*`, `lambda:*`, `rds:*`, `route53:*`, `cloudfront:*` on
`Resource: "*"`. So a plan-phase token — which runs unattended on every push — can assume its way
to full write in every member account. The claim below that a compromised plan token "can never
mutate real infrastructure, full stop" is **not true today**. Fixing it properly needs a second,
read-only member role vended by the same StackSet and a way for the pipeline to pass a different
role ARN per phase, which is a change across `github-automation` and every consuming repo — not a
one-line fix, and not yet done.

## The split: plan / apply / module-skills

Every repo's GitHub Actions pipeline authenticates via OIDC into one of three narrow roles,
never one broad combined role:

| Role | Purpose | Trust condition | Permissions |
|---|---|---|---|
| **plan** | `terraform plan` — read-only, AND every job's `get-aws-secrets` call (see below) | multiple subjects, see next section — not just `ref:refs/heads/main` | AWS-managed `ReadOnlyAccess` + a narrow custom policy: state-lock S3 object R/W (not real state writes — just the native S3 lock file), `kms:Decrypt` on the state key, `sts:AssumeRole` on the member CICD role pattern only, audit-log S3 write to one prefix, `sts:GetCallerIdentity`, and `secretsmanager:GetSecretValue`/`DescribeSecret` scoped to `secret:github/{org}/*` (AWS's `ReadOnlyAccess` deliberately excludes secret *values*) |
| **apply** | `terraform apply` — write, real infrastructure mutation | `repo:{owner}/{repo}:ref:refs/heads/main` — see next section for why this isn't environment-scoped | Full write access to the specific AWS services that repo's Terraform manages — scoped per-repo, not blanket `*:*` |
| **module-skills** (optional) | Non-infra pipelines (test runners, Claude skills, analytics) | `repo:{owner}/{repo}:environment:*` or wildcard subjects, only created when actually needed | Audit-log write + KMS for that write + caller-identity only — zero state/IAM access, smallest possible blast radius |

**Why a role split, not one role for everything**: separation of duties. The intent is that a
compromised or misconfigured plan-phase token — which runs on every push, unattended — can read
state and secrets but never write, independent of whatever the approval-gate story is for a given
billing tier.

> ⚠️ **That intent is not yet realised.** The plan role's own IAM policy can't write, but it *can*
> `sts:AssumeRole` into the member CICD role, which can. See "Status, 2026-07-29" at the top of
> this file. Treat the plan role as write-capable in every member account until that's fixed.

## The real OIDC subject list — derived empirically, not from first principles

**`get-aws-secrets` always authenticates as the *plan* role**, unconditionally, in every reusable
workflow that calls it (checkov-scan, quality-gate, plan-encrypt, apply-decrypt, approve-gate,
audit) — regardless of which role that job's real Terraform action ultimately needs. This one fact
drives the whole subject list, and it's easy to get wrong by reasoning from the job names instead
of testing each stage for real:

```
github_oidc_plan_subjects = [
  "repo:{owner}/{repo}:ref:refs/heads/main",       # checkov-scan, quality-gate, apply-decrypt,
                                                     # audit - none of these set `environment:`
  "repo:{owner}/{repo}:environment:{org}",          # plan-encrypt sets `environment: ${{ inputs.org }}`
  "repo:{owner}/{repo}:environment:{org}-approve",  # approve-gate sets `environment: ${{ inputs.apply_env }}`,
                                                     # and reusable-tf-parse-config.yaml computes
                                                     # apply_env as "${org}-approve" - use that exact
                                                     # value, not a hand-picked environment name
]

github_oidc_apply_subjects = [
  "repo:{owner}/{repo}:ref:refs/heads/main",  # apply-decrypt's REAL terraform-apply step
                                                # (a second, separate AssumeRole using the
                                                # oidc_apply_role_arn secret) - this job
                                                # deliberately sets no `environment:`, so its
                                                # sub claim is the plain ref, not an
                                                # environment-scoped one
]
```

A tempting-but-wrong mental model is "plan role trusts the plan ref, apply role trusts the approve
environment" — that's not what the code actually does. Verify against the real reusable workflow
files in `hoad-org/github-automation`, not this description, if they've changed since this was
written: search each `.github/workflows/reusable-tf-*.yaml` for `environment:` at the job level.

## GitHub Environment protection at $0 — a real, hard billing limitation

GitHub's environment protection rules — **both** required reviewers **and** wait timer — require a
paid plan (GitHub Pro/Team/Enterprise) on a *private* repository. Confirmed live: both return
`"Please ensure the billing plan supports the ... protection rule"` (HTTP 422) on this org's actual
free-tier billing. This means the `{org}-approve` GitHub Environment **cannot** enforce a real
mid-run pause for human review at $0 cost — despite what earlier documentation (and the reference
this design was adapted from, which assumes a paid plan) implied.

**The $0-tier substitute gate**: `deploy.yaml` triggers on `workflow_dispatch` only — deliberately
no `push: branches: [main]` trigger. A merge to `main` does *not* automatically deploy; someone
with write access must explicitly start the run. That's real (only repo write-access holders can
invoke `workflow_dispatch`) but weaker than a true mid-run pause — once started, the run proceeds
through Plan → Approve → Apply with no way to stop and review the plan output first inside that
same run. Compensating control: `pr-validate.yml` runs the same Trivy/Checkov/Quality-Gate/Plan
gates on every PR *before* merge, so by the time someone triggers `workflow_dispatch` the plan has
already been seen once. See [SECURITY_ROADMAP.md](SECURITY_ROADMAP.md) for the upgrade once
there's a paid plan.

## Single source of truth: `reusable-tf-parse-config.yaml`

Don't hardcode `aws_region`/`aws_account_id`/`partition`/`apply_env` as literal strings in a
`deploy.yaml`'s job `with:` blocks — every one of those is a duplicate of a value that already
lives in `configs/orgs/{org}.tfvars`, and duplicated values drift (this repo's own `deploy.yaml`
did exactly that for its first several revisions). Add a `plan_parse-config` job as the first job
in the workflow (`uses: .../reusable-tf-parse-config.yaml@main`, `with: {org: <org>}`), and have
every other job read `needs.plan_parse-config.outputs.*` instead. This is the pattern the
reference `deploy-orchestrated.yml` this design is adapted from uses throughout.

## Naming formula (load-bearing — must match exactly)

```
{org}-{region_short}-{system}-github-oidc-tf-plan-role
{org}-{region_short}-{system}-github-oidc-tf-apply-role
{org}-{region_short}-{system}-github-oidc-module-skills-role
```

No `partition_short` segment — deliberately different from the general resource-naming formula
in [ARCHITECTURE.md](ARCHITECTURE.md). `hoad-org/github-automation`'s `get-aws-secrets` action
derives the plan-role ARN mechanically from exactly this formula
(`{org}-{region_short}-platform-github-oidc-tf-plan-role`, `system` hardcoded as `platform` in
that action today). If you rename the role, `get-aws-secrets` breaks silently until the action is
updated to match.

## Two deployment models — management-account roles vs. workloads-account roles

**Management-account roles** (created by the seed repo's own `seed-terraform/main.tf`): used by
repos whose Terraform targets the management account directly — the seed repo itself,
`aws-accounts`, `aws-baselines`, `aws-org`.

**Workloads-account roles**: each workloads account gets its OWN plan/apply/module-skills roles,
created either by (a) the seed repo's CloudFormation StackSet deploying a **member CICD role**
into every workloads account (see below), assumed cross-account from the management-account roles
— the "two-hop" model — or (b) a workloads-account repo creating its own roles directly in that
account via a one-hop OIDC provider — used when a repo doesn't need the shared state backend's
cross-account complexity. Prefer (a) for anything that should be manageable centrally; (b) is an
escape hatch, not the default.

## The member CICD role (StackSet-deployed)

A CloudFormation StackSet (`SERVICE_MANAGED` permission model, `auto_deployment.enabled = true`)
deploys a single IAM role — `{name_prefix}-cicd-role` — into **every** account in the target OUs
(`Workloads/Prod`, `Workloads/NonProd`, plus `Management`/`Security`/`Infrastructure` as needed).
This is what lets the landing zone "deploy to mgmt alone, or any member account alone, or all of
them" — the StackSet handles fan-out; each individual repo's Terraform only ever needs to know
its own target account.

**A real bug found and fixed in this template**: the role's `AssumeRolePolicyDocument` originally
trusted only `Principal: {Service: "cloudformation.amazonaws.com"}` — meaning nothing except the
CloudFormation service itself could ever assume it, defeating its entire purpose as a cross-account
CI/CD execution role. Fixed to trust the calling management-account OIDC role's ARN
(`AWS: {management_role_arn}`) instead, with `aws:PrincipalOrgID` as defense-in-depth.

**Prerequisites for the StackSet to deploy at all** (both required, commonly conflated as one):
1. `aws organizations enable-aws-service-access --service-principal=member.org.stacksets.cloudformation.amazonaws.com`
2. `aws cloudformation activate-organizations-access` — a **separate** CloudFormation-specific
   switch. Both must show enabled (`aws cloudformation describe-organizations-access`) before
   `CreateStackSet` with `permission_model = SERVICE_MANAGED` will succeed.

**Status: live and confirmed working (2026-07-15).** The stack set instance had been sitting
tainted since an earlier failed attempt — the seed repo's own first real `deploy.yaml` run through
the full CI pipeline (Trivy → Checkov → Quality Gate → Plan → Apply) replaced it for real (`1
added, 0 changed, 1 destroyed`). Confirmed live: `hcp-cmc-euw1-platform-member-role-cicd-role`
exists in `hcp-craighoad-prod` (624426145233) and is assumable from
`hcp-cmc-euw1-platform-oidc-role` (the management-account workloads role) — the two-hop chain
described above is no longer theoretical, a real repo (`personal-ai-cloud`) is bootstrapping
against it as of this writing.

**Target OU IDs must be real, live-queried values** — see the corrected list in
[ARCHITECTURE.md](ARCHITECTURE.md). A previous attempt used fabricated OU IDs that didn't exist
in the Organization at all, which fails with `Target organizational unit could not be found`
rather than any clearer error.
