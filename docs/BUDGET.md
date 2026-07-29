# Budget — $0/month, enterprise-lite

This landing zone runs on a strict **$0/month** budget. "Enterprise-lite" means real separation of
duties, real least-privilege IAM, a real multi-account structure, a real centralized Terraform
backend — at zero AWS spend, not "cheap." Nothing in this estate should ever incur cost without an
explicit, informed decision. This file is the source of truth for what's safe to provision and
what isn't.

**If you are a Claude session about to provision a new AWS resource type: check this file first.
If the service/action isn't in the "in active use" list below, don't assume it's free — ask.**

## Real, measured spend — 2026-07-29

Live `aws ce get-cost-and-usage`, 1–29 July 2026. **$21.44 actual**, Cost Explorer forecasting
**$46.71**. `platform-monthly-budget` ($5 limit, platform accounts only) was at **$5.75 — already
breached**. The estimates previously in this file were substantially low.

| Service | July MTD | Attribution |
|---|---|---|
| AWS WAF | $7.22 | terrorgems-prod — **orphaned, deleted 2026-07-29** |
| AmazonCloudWatch | $3.96 | `DashboardsUsageHour-Basic`, all 6 member accounts — **deleted 2026-07-29** |
| KMS | $3.59 | $2.70 terrorgems (3 CMKs), $0.90 mgmt (state key) |
| Tax | $3.58 | |
| Secrets Manager | $1.89 | 3 terrorgems + 2 × `github/hcp/config` |
| Route 53 | $1.02 | craighoad.com + terrorgems.com zones |
| S3 / API GW / CE API | $0.18 | |

**Two findings worth naming, because both contradicted this file:**

1. **`shared-waf-for-static-websites`** (CLOUDFRONT scope, us-east-1, 3 rules, terrorgems-prod) was
   **attached to nothing** — `list-resources-for-web-acl` empty, and the terrorgems.com
   distribution's `WebACLId` blank. It existed in no Terraform file anywhere. $7.22/month
   (~$87/year) protecting nothing. This file previously listed WAF as legitimate TerrorGems product
   spend; it wasn't. **Deleted.**
2. **The cost-tracking module's own CloudWatch dashboard was the largest platform-side cost line.**
   `hcp-platform-cost-overview` was billed *per account it was applied to*, not once centrally —
   $1.46 + $0.58 + $0.48 × 4 = ~$3.96/month, for two widgets showing `AWS/Billing
   EstimatedCharges`, which Cost Explorer displays for free. **Deleted from all six accounts and
   removed from the module** (`aws-terraform-platform-aws-baselines` PR #9). Do not re-add a
   dashboard to a module that fans out across every vended account.

Post-cleanup run rate is roughly **$3/month** (state CMK $0.90, two config secrets $0.80, two
hosted zones $1.02, S3 pennies) plus ~$15/domain/year in registrar renewals. Remaining reducible
items, none yet actioned: two terrorgems CMKs sharing the identical description `KMS key for
DynamoDB encryption in prd environment` (one likely orphaned, ~$1/month), and the `gcp-gemini-key`
/ `youtube-api-key` secrets in terrorgems (~$0.80/month, `asatst`-era leftovers alongside a dead
Step Functions estate).

**Scope: this applies to the platform/landing-zone itself and its shared accounts (management,
security, infrastructure) and to `personal-ai-cloud`** — not retroactively to already-established
products with their own separate cost model. `hcp-terrorgems-prod` (TerrorGems) is a real product
with its own already-decided, phased cost plan and currently runs real spend (WAF, KMS,
Secrets Manager — confirmed via a live Cost Explorer check: ~$4.55/month WAF + ~$1.70/month KMS +
~$0.68/month Secrets Manager, entirely attributable to that one account). That's expected and
out of scope for this file — don't "fix" TerrorGems' spend as if it were a platform violation.

## In active use today (all free-tier-safe at current scale)

Lambda, API Gateway, S3, CloudFront, DynamoDB, SNS, SQS, IAM, STS, CloudFormation (including
StackSets — the service itself is free, only the resources it deploys can cost money), Route53,
ACM (public certs are free), native S3 conditional-writes state locking (`use_lockfile = true` —
no DynamoDB lock table, avoiding that cost entirely), AWS Organizations (SCPs are free), AWS
Budgets (first 2 budgets/account free), IAM Identity Center / SSO (free), CloudTrail (management
events are free; data events cost — not currently enabled).

**Two real, named, small-but-nonzero exceptions** — not free, kept anyway, documented so nobody
re-discovers them by surprise in a Cost Explorer bill:

- **KMS (customer-managed key)** — see below.
- **AWS Secrets Manager** — ~$0.40/secret/month, **not** free-tier-eligible. Kept to one secret
  per org per account (`github/{org}/config`, a JSON blob — see [SECRETS.md](SECRETS.md)) rather
  than one secret per value, after finding the original 6-secrets-per-org design would have cost
  6x that. Real management-account spend confirmed via Cost Explorer:
  ~$0.23/month today (partial-month, will settle around $0.40/month once fully billed for the one
  consolidated secret) — small, accepted, not eliminated (SSM Parameter Store's free tier is an
  option for non-sensitive values, not pursued this round since the consolidation already got the
  real cost down to a level not worth the extra architectural complexity).

## The one named, accepted exception: customer-managed KMS key

The shared Terraform state bucket (`hcp-cmc-euw1-platform-tfstate-prd`) is encrypted with a
**customer-managed** KMS key (`alias/hcp-cmc-euw1-platform-tfstate`), not AWS-managed SSE-S3
(free). A customer-managed key costs **~$1/month**. This is a real, known, deliberate exception —
not an oversight.

**Why it's kept, not switched to free SSE-S3**: the blast radius of switching is real.
`personal-ai-cloud`'s IAM policies (`infra/aws/environments/prod/oidc.tf`,
`infra/aws/configs/envs/prd.tfvars`) already hardcode `Resource` ARN statements pointing at this
exact key — committed Terraform source, not yet applied. Switching the bucket's encryption would
leave those statements dangling without a coordinated follow-up across at least that repo. If this
is ever revisited, grep the whole estate for `Resource = [var.shared_kms_key_arn]`-shaped
statements first — don't assume this is the only dependent.

## Guardrails — enforced at the IAM/billing layer, not just in prose here

**Two AWS Budgets are live** (`aws budgets describe-budgets --account-id 395101865577`):
- `global-monthly-budget` — pre-existing, $50/month limit across the whole org, alerts at 90%
  actual / 100% forecasted, email to `craighoad+billing@hotmail.com`. Covers everything including
  `hcp-terrorgems-prod`'s legitimate product spend.
- `platform-monthly-budget` — created this session, **$5/month limit, scoped by `LinkedAccount`
  cost filter to just the platform accounts** (management, hcp-audit, hcp-log-archive,
  hcp-shared-services, hcp-craighoad-prod, hcp-qa — deliberately excludes
  `hcp-terrorgems-prod`), alerts at 80% actual / 100% forecasted, same email. This is the one that
  actually watches the $0 goal in this file — real spend at creation time was $1.31/month actual,
  $2.15/month forecasted (see the Secrets Manager section above for what that is).

A Service Control Policy denies the specific, well-known-costly services/actions below at the
**Workloads OU** (Prod + NonProd) — deliberately not at root, so Management/Security/Infrastructure
stay unrestricted for baseline tooling and for administering the SCP itself. This is the real
technical enforcement layer: even a future Claude session that reads this file wrong, or doesn't
read it at all, cannot actually provision these resources — the API call itself is denied.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyKnownCostlyComputeAndDatabase",
      "Effect": "Deny",
      "Action": [
        "rds:CreateDBInstance", "rds:CreateDBCluster", "redshift:CreateCluster",
        "es:CreateElasticsearchDomain", "es:CreateDomain", "opensearch:CreateDomain",
        "eks:CreateCluster", "sagemaker:CreateNotebookInstance", "sagemaker:CreateEndpoint",
        "sagemaker:CreateTrainingJob", "kafka:CreateCluster", "elasticache:CreateCacheCluster",
        "elasticache:CreateReplicationGroup", "memorydb:CreateCluster"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyNATGatewayCosts",
      "Effect": "Deny",
      "Action": ["ec2:CreateNatGateway"],
      "Resource": "*"
    },
    {
      "Sid": "DenyLargeAndGPUInstanceTypes",
      "Effect": "Deny",
      "Action": ["ec2:RunInstances"],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "ForAnyValue:StringNotLike": {
          "ec2:InstanceType": ["t2.micro", "t2.small", "t3.micro", "t3.small", "t4g.micro", "t4g.small"]
        }
      }
    },
    {
      "Sid": "DenyGuardDutyAndConfigEnable",
      "Effect": "Deny",
      "Action": [
        "guardduty:CreateDetector", "config:PutConfigurationRecorder",
        "config:StartConfigurationRecorder", "securityhub:EnableSecurityHub", "macie2:EnableMacie"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyManagedFileSystemsAndWorkspaces",
      "Effect": "Deny",
      "Action": [
        "fsx:CreateFileSystem", "elasticfilesystem:CreateFileSystem",
        "workspaces:CreateWorkspaces", "globalaccelerator:CreateAccelerator"
      ],
      "Resource": "*"
    }
  ]
}
```

Deliberately does **not** touch anything in active use above, or free-tier-eligible EC2 instance
types — a $0 budget with zero EC2 at all would be more restrictive than this estate actually needs;
`t2`/`t3`/`t4g` micro/small stay allowed.

`guardduty:CreateDetector`/`config:PutConfigurationRecorder`/etc. matter here specifically as a
guardrail against a **future Claude session "helpfully" enabling them** — see
[SECURITY_BASELINE.md](SECURITY_BASELINE.md) for why they're off by design, and
[SECURITY_ROADMAP.md](SECURITY_ROADMAP.md) for when to turn them on.

**This SCP is not managed by Terraform** — it was created by a raw `aws organizations
create-policy` call and is absent from `aws-terraform-platform-aws-org`'s `var.scps` map. It is
therefore not in any state file, will not be recreated if deleted, and no drift check will notice
if it disappears. Confirmed live 2026-07-29: `p-5i30zmvq hcp-cost-guardrail` exists and is attached
to `ou-4if6-on6pmizj` (Workloads/Prod) only. The entire $0 enforcement layer is click-ops. Adopting
it into `var.scps` is outstanding work.

**Status of this SCP**: `aws organizations create-policy`, policy ID `p-5i30zmvq`.
`aws iam simulate-custom-policy` — the natural tool to test this against real role policies before
attaching — **doesn't actually support Service Control Policies at all** (a real AWS limitation,
confirmed live: it rejects any SCP-shaped document, even a single-statement one, with
`InvalidInput`; the IAM Policy Simulator is built for identity/resource-based policies only, there's
no official SCP simulator). Verified manually instead: none of the real, deployed role policies in
this estate (the seed repo's `workloads_oidc_policy`, `personal-ai-cloud`'s `apply_policy`) grant
any action this SCP denies — the closest overlap is `workloads_oidc_policy`'s broad `ec2:*`, which
this SCP narrows (blocking NAT Gateway creation and non-free-tier instance types specifically)
without breaking anything currently exercised, since nothing in this estate actually runs EC2
instances today (it's Lambda-based).

**Attached to `Workloads/Prod` (`ou-4if6-on6pmizj`)**, confirmed live via
`aws organizations list-policies-for-target`. **Not yet attached to `Workloads/NonProd`**
(`ou-4if6-mjcqxg7t`) — staged rollout, Prod-first: attach to NonProd once a real CI cycle has
succeeded against `hcp-craighoad-prod`/`hcp-terrorgems-prod` under this SCP, confirming no
unexpected collision in practice, not just on paper.
