# IaC — Terraform / CDK / Pulumi

Deep-dive for reviewing infrastructure-as-code. IaC is powerful because one `apply` can change your entire estate — which is exactly why a bad line can destroy your production database, leak your state file, or open an IAM role to the world. The reviewer's job is to read the **diff for blast radius**: what does this create, change, and — most dangerously — **replace or destroy**? Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it causes.

Examples are Terraform-first; the same hazards (state, drift, destructive replaces, broad IAM, unpinned providers) apply to CDK, Pulumi, and CloudFormation. (SQL schema *content* → **database-review**; secret-handling depth → **security-review**.)

---

## 1. Local / unlocked / unencrypted state

State is the source of truth that maps your config to real resources — and it contains every attribute, including secrets. Local state can't be shared safely; unlocked state lets two applies corrupt it; unencrypted state leaks secrets at rest.

❌ No backend (local state) or one without locking/encryption:
```hcl
# no backend block → terraform.tfstate sits on someone's laptop / in the repo  ❌
terraform {
  backend "s3" {
    bucket = "tf-state"
    key    = "prod.tfstate"        # ❌ no dynamodb_table (no lock), no encrypt
  }
}
```
✅ Remote backend with locking + encryption:
```hcl
terraform {
  backend "s3" {
    bucket         = "tf-state"
    key            = "prod/terraform.tfstate"
    dynamodb_table = "tf-locks"    # ✅ state lock — serializes applies
    encrypt        = true          # ✅ encrypted at rest
    kms_key_id     = "alias/tf-state"
  }
}
```
> Incident: two engineers `apply` the same workspace at once with no lock; the writes interleave and the state file is corrupted — now Terraform's view of reality is wrong and the next plan proposes to destroy resources that actually exist. Locking serializes applies; remote+encrypted state means a stolen laptop or a public bucket doesn't leak your DB passwords. (Newer Terraform supports native S3 state locking; older setups need the DynamoDB table.)

---

## 2. Secrets in state or plaintext `tfvars`

Many resources write secrets into state (RDS passwords, generated keys), and committed `*.tfvars` are just plaintext secrets in git.

❌ Secret in a committed var file or hardcoded:
```hcl
# prod.tfvars (committed)  ❌
db_password = "Pa55w0rd!"
resource "aws_db_instance" "db" {
  password = "Pa55w0rd!"          # ❌ also lands in state in cleartext
}
```
✅ Source secrets from a manager; keep var files out of git:
```hcl
data "aws_secretsmanager_secret_version" "db" { secret_id = "prod/db" }
resource "aws_db_instance" "db" {
  password = data.aws_secretsmanager_secret_version.db.secret_string  # still hits state →
}
# → so: encrypt state (§1), restrict who can read it, and gitignore *.tfvars
```
> Incident: a `prod.tfvars` with the database password is committed to a repo dozens of people can read; or the password lands in an unencrypted state file in a public bucket. Either is a breach requiring rotation. Treat `*.tfvars` like secrets (gitignore them), pull real secrets from a manager, and accept that anything in state must be encrypted and access-controlled. (Secret-store wiring depth → **security-review**.)

---

## 3. `apply` on push — no `plan` review in CI

The plan is the review artifact: it shows exactly what will change before it changes. Auto-`apply` on push removes the one chance to catch a destructive diff.

❌ Pipeline applies automatically on merge:
```yaml
- run: terraform apply -auto-approve     # ❌ whatever the diff is, it ships
```
✅ Plan on PR, apply behind a gate after the plan is read:
```yaml
# on pull_request: terraform plan -out=tfplan  → post the plan to the PR for review
# on merge, in a gated/approved environment:
- run: terraform apply tfplan            # ✅ applies the EXACT reviewed plan
```
> Incident: a refactor that looks innocuous in the HCL actually forces replacement of a stateful resource (see §5). With `-auto-approve` on merge it executes before anyone sees the plan, and prod is gone. Posting `terraform plan` on the PR makes the destruction visible — `# forces replacement` / `-/+ destroy and then create` — so a human catches it. Always apply a saved plan, not a fresh one, so what's reviewed is what runs.

---

## 4. Unpinned providers and modules

Providers and modules are code that runs against your infra. An unpinned version means an `init` can pull a new major with breaking or destructive behavior changes.

❌ Floating versions:
```hcl
terraform { required_providers { aws = { source = "hashicorp/aws" } } }   # ❌ no version
module "vpc" { source = "terraform-aws-modules/vpc/aws" }                  # ❌ latest
```
✅ Pin with constraints + a committed lockfile:
```hcl
terraform {
  required_version = "~> 1.7"
  required_providers { aws = { source = "hashicorp/aws", version = "~> 5.40" } }
}
module "vpc" { source = "terraform-aws-modules/vpc/aws", version = "5.8.1" }   # exact
# commit .terraform.lock.hcl so provider hashes are pinned and verified
```
> Incident: `terraform init` on a teammate's machine pulls AWS provider v6 (a major bump) whose changed defaults rewrite a security group or rename an attribute, and the next apply proposes destructive churn nobody intended. Pin provider/module versions and commit `.terraform.lock.hcl` so every machine and every CI run resolves the same, verified versions.

---

## 5. Destructive replace of a stateful resource

The most dangerous IaC diff: a change that **forces replacement** (`-/+`) of a resource that holds data — a database, disk, volume, bucket. Terraform destroys the old one and creates a new empty one. For stateful resources this is irreversible data loss.

❌ Changing a force-new attribute on a stateful resource:
```hcl
resource "aws_db_instance" "db" {
  identifier = "prod-db-v2"     # ❌ changing identifier forces a REPLACE → destroy prod data
  # (also: changing availability_zone, engine in some cases, kms_key, etc.)
}
```
✅ Modify in place where possible; protect against accidental replace:
```hcl
resource "aws_db_instance" "db" {
  identifier = "prod-db"        # don't churn force-new attributes
  lifecycle {
    prevent_destroy = true      # ✅ apply FAILS rather than destroying this resource
    # ignore_changes = [...] for attributes managed out-of-band
  }
}
```
> Incident: someone renames the RDS `identifier` (or flips a `force_new` attribute) in a "cleanup" PR; the plan shows `-/+ destroy and then create`, but with auto-apply or a rubber-stamp review it runs, **deleting the production database and standing up an empty one** — and unless a final snapshot existed, the data is gone. Read every `# forces replacement` in the plan; put `prevent_destroy = true` on every stateful resource so even a mistaken replace fails closed; ensure deletion protection / final snapshots are on.

---

## 6. `count` where `for_each` belongs

`count` indexes resources by position. Remove or reorder an element in the middle of the list and every later resource shifts index — so Terraform destroys and recreates them.

❌ Positional `count` over an identity-bearing list:
```hcl
resource "aws_iam_user" "u" {
  count = length(var.usernames)        # ❌ indexed by position
  name  = var.usernames[count.index]   # remove usernames[0] → every other user recreated
}
```
✅ `for_each` keyed by a stable identity:
```hcl
resource "aws_iam_user" "u" {
  for_each = toset(var.usernames)      # ✅ keyed by name, not position
  name     = each.key                  # remove one → only that one is destroyed
}
```
> Incident: an ops change drops one name from the middle of a `count`-based list of database users / DNS records / queues; because everything after it shifts index, Terraform destroys and recreates a dozen unrelated stateful resources, causing an outage. Use `for_each` (map/set keyed by identity) for anything where adding/removing one element shouldn't disturb the others. Reserve `count` for "N identical, fungible" or simple on/off toggles.

---

## 7. Overly broad IAM policies — `Action`/`Resource` = `*`

IaC is where most over-permissioning is born. A policy with `"Action": "*"` on `"Resource": "*"` is admin; one leaked credential with that policy attached is a full account compromise.

❌ Wildcard everything:
```hcl
resource "aws_iam_policy" "app" {
  policy = jsonencode({ Statement = [{
    Effect = "Allow", Action = "*", Resource = "*"   # ❌ god-mode
  }]})
}
```
✅ Scope action and resource to what the workload needs:
```hcl
resource "aws_iam_policy" "app" {
  policy = jsonencode({ Statement = [{
    Effect   = "Allow",
    Action   = ["s3:GetObject", "s3:PutObject"],            # specific verbs
    Resource = ["arn:aws:s3:::app-uploads/*"]               # specific bucket
  }]})
}
```
> Incident: an app role is granted `*:*` "to ship faster"; later that role's credentials leak (a logged token, an SSRF to the metadata endpoint), and the attacker can now read every bucket, delete every database, and create IAM users — because the role was admin. Least-privilege at the IaC layer caps the blast radius of any future credential leak. Avoid `*` in both `Action` and `Resource`; scope by ARN.

---

## 8. Missing tags / ownership

Untagged resources are invisible in cost reports, hard to attribute during an incident, and easy to orphan. Tagging is a guardrail, not cosmetics.

❌ No tags / inconsistent tags:
```hcl
resource "aws_instance" "worker" { ami = "..." instance_type = "m5.large" }  # ❌ who owns this? what cost center?
```
✅ Enforce a tag baseline (provider default_tags):
```hcl
provider "aws" {
  default_tags { tags = { team = "payments", env = "prod", managed_by = "terraform", cost_center = "cc-42" } }
}
```
> Incident: a cost spike appears and nobody can tell which team or service owns the runaway fleet, because the instances are untagged; or an orphaned, untagged resource runs for months unnoticed on the bill. Tags make resources attributable for cost, ownership, and incident response. Set `default_tags` on the provider so every resource inherits the baseline.

---

## 9. Data-source / dependency races

Terraform parallelizes by default, ordering only by detected dependencies. A `data` source or resource that reads something another resource is still creating can race — reading a stale or absent value.

❌ Implicit ordering that can read too early:
```hcl
data "aws_instances" "web" { filter { name = "tag:role" values = ["web"] } }  # ❌ may run before instances exist
resource "aws_lb_target_group_attachment" "a" {
  count            = length(data.aws_instances.web.ids)   # 0 on first apply → no attachments
  target_id        = data.aws_instances.web.ids[count.index]
}
```
✅ Reference the resource directly so the dependency is explicit:
```hcl
resource "aws_lb_target_group_attachment" "a" {
  for_each         = aws_instance.web        # ✅ real dependency → correct ordering
  target_group_arn = aws_lb_target_group.tg.arn
  target_id        = each.value.id
}
```
> Incident: a `data` source enumerating instances by tag runs before those instances are created, so on a fresh apply it returns zero and the load-balancer attachments aren't made — the new environment comes up with nothing wired to the LB and serves 503s. Prefer referencing resource attributes directly (which Terraform tracks as a dependency) over `data` lookups of things you're creating in the same run; use `depends_on` only when a dependency is genuinely implicit.

---

## 10. Drift and out-of-band changes

When someone changes infra in the console (or another tool) instead of through IaC, real state drifts from the code. The next `apply` may revert the fix — or, worse, a `terraform import` / `ignore_changes` was used to paper over it and now the code lies.

❌ Manual change + Terraform unaware:
```text
# someone bumps the RDS instance class in the console during an incident  ❌
# next terraform apply proposes to REVERT it back to the smaller class → re-outage
```
✅ Detect drift in CI; make IaC the only writer:
```yaml
- run: terraform plan -detailed-exitcode   # ✅ exit 2 = drift; fail the build / alert
# then reconcile in code (update HCL to match reality) — don't ignore_changes the truth away
```
> Incident: an on-call engineer scales up a database in the console to survive a traffic spike; days later a routine `terraform apply` of an unrelated change quietly reverts the instance back to the small class, and the outage returns at the worst time. Run scheduled `terraform plan -detailed-exitcode` to surface drift, treat IaC as the single source of truth, and reconcile real changes back into code rather than hiding them with `ignore_changes`.

---

## Quick scan checklist

- [ ] State uses a **remote backend** with **locking** and **encryption at rest** — no local/committed state.
- [ ] No secrets in committed `*.tfvars`; secrets sourced from a manager; state access-controlled because it holds them.
- [ ] CI runs `plan` on the PR for review; `apply` is gated and applies the **saved plan**, not `-auto-approve` on push.
- [ ] Providers, modules, and `required_version` are **pinned**; `.terraform.lock.hcl` is committed.
- [ ] The plan was read for **`# forces replacement`**; stateful resources carry `lifecycle { prevent_destroy = true }` + deletion protection.
- [ ] `for_each` (keyed by identity) is used instead of `count` wherever element add/remove shouldn't recreate others.
- [ ] IAM policies scope `Action` and `Resource` — no `"*"`/`"*"` admin grants.
- [ ] Resources are tagged (owner/env/cost-center), ideally via provider `default_tags`.
- [ ] Dependencies are expressed by direct resource references, not `data` lookups that can race on a fresh apply.
- [ ] Drift is detected in CI (`plan -detailed-exitcode`); IaC is the single writer, with real changes reconciled into code.
