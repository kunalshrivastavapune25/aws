# AWS Firewall Manager to CloudFormation Migration Guide

## End-to-End Implementation Runbook for Migrating a Manually Built Firewall Manager Environment to Infrastructure as Code

**Document version:** 1.0
**Date:** July 2026
**Audience:** AWS engineers familiar with AWS basics, new to Firewall Manager and CloudFormation for Firewall Manager
**Scope:** AWS Organization (~44 accounts), dedicated Security Infra account as Firewall Manager delegated administrator, environment built manually via the AWS Console, no existing IaC or LLD. The live environment is the only source of truth.

---

## How to Use This Document

This guide takes you from beginner-level concepts (Part 1) through hands-on discovery (Parts 2 and 5), planning (Part 3), CloudFormation design (Part 4), deployment (Part 6), enterprise best practices (Part 7), and reference diagrams (Part 8). It is written to be executed top-to-bottom as a production migration runbook, but each part is self-contained enough to be used as a reference chapter.

Two golden rules apply throughout:

1. **Never delete a live protection before its IaC replacement is verified.** The migration approach in this guide is "model, import or shadow-deploy, verify, then cut over" — not "tear down and rebuild."
2. **Firewall Manager owns what Firewall Manager created.** Resources that FMS auto-created in member accounts (Web ACLs named `FMManagedWebACLV2...`, auto-created Network Firewall endpoints, etc.) must never be imported into your own CloudFormation stacks. You model the *policy* in CloudFormation; FMS continues to own the distributed artifacts.

---
# Part 1 – Conceptual Architecture

## 1.1 The Building Blocks, in Plain Language

**AWS Organizations.** The service that groups many AWS accounts into a single hierarchy so they can be governed centrally (consolidated billing, policies, delegated administration). Your ~44 accounts all belong to one Organization.

**Management Account.** The single account that owns the Organization. It is the "root of trust": it creates/invites accounts, defines Organizational Units, and *delegates* administration of specific services (like Firewall Manager) to other accounts. Best practice is to run almost nothing in it.

**Delegated Administrator.** A member account that the management account has authorized to administer one service on behalf of the whole Organization. Your **Security Infra account is the Firewall Manager delegated administrator**: it is where FMS policies are created and where compliance is monitored, without needing to log in to the management account.

**AWS Firewall Manager (FMS).** A central *orchestrator*. It does not inspect traffic itself. You define a **security policy** once in the delegated admin account, define its scope (which accounts/OUs, which resource types, which tags), and FMS automatically creates, associates, and maintains the underlying protections (WAF Web ACLs, Network Firewalls, security groups, Shield protections, etc.) inside every in-scope member account — including accounts created in the future.

**AWS WAF.** A web application firewall that inspects HTTP(S) requests at layer 7 and allows, blocks, counts, or challenges them based on rules (SQL injection patterns, IP reputation, rate limits, geo match, etc.). WAF attaches to CloudFront distributions, Application Load Balancers, API Gateway REST APIs, AppSync, Cognito user pools, App Runner, and Verified Access.

**AWS Network Firewall (if applicable).** A managed layer 3/4 (+ deep packet inspection) network firewall for VPC traffic. It filters traffic flowing through VPC subnets using stateless and stateful rules (domain lists, Suricata rules, 5-tuple rules). FMS can deploy Network Firewalls into member VPCs automatically.

**Rule.** The smallest decision unit. In WAF, a rule is "if the request matches *this condition*, take *this action* (Allow / Block / Count / CAPTCHA / Challenge)". In Network Firewall, a rule is a stateless or stateful match (e.g., "drop outbound TCP 445", or a Suricata signature).

**Rule Group.** A named, reusable container of rules. Two flavors in WAF: **AWS Managed Rule Groups** (maintained by AWS/Marketplace, e.g., `AWSManagedRulesCommonRuleSet`) and **custom rule groups** you author. Rule groups have a fixed WCU (capacity) budget in WAF. Network Firewall likewise has stateless and stateful rule groups with a capacity setting.

**Web ACL (Web Access Control List).** The top-level WAF object that is actually *attached* to a resource (ALB, CloudFront, etc.). A Web ACL contains an ordered list of rules and rule-group references, a default action (Allow or Block), and a WCU capacity limit (up to 5,000 by default). One resource can have exactly one Web ACL; one Web ACL can protect many resources.

**Firewall Policy (Network Firewall).** The Network Firewall equivalent of a Web ACL: it bundles stateless rule groups, stateful rule groups, and default actions, and is referenced by one or more Firewall endpoints in VPCs.

**Firewall Manager Security Policy.** The central FMS object (created in the Security Infra account) that says: "For accounts/OUs X, resource types Y, and tags Z, enforce protection P" — where P is a WAF configuration, a Network Firewall configuration, a Shield Advanced protection, a security-group policy, etc. FMS continuously reconciles reality against this policy.

**Resource Associations.** The link between a protection and the thing it protects — e.g., a Web ACL *associated* with an ALB, or a Network Firewall endpoint placed into a subnet with routes pointing at it. FMS creates and repairs these associations automatically for in-scope resources.

**Organizational Units (OUs).** Folders in the Organization tree used to group accounts (e.g., `Prod`, `NonProd`, `Sandbox`). FMS policy scope is typically expressed in OUs, so new accounts landing in an OU are protected automatically.

**Member Accounts.** All non-management accounts in the Organization — your ~44 workload accounts. They *receive* protections from FMS; their local admins generally should not (and via SCPs often cannot) modify FMS-managed resources.

## 1.2 How the Pieces Relate

```
AWS Organization (Management Account)
 └── delegates FMS administration to → Security Infra Account (Delegated Admin)
       └── Firewall Manager Security Policy
             ├── Scope: OUs / account list / tags / resource types / regions
             └── Security service definition:
                   WAF:  Rule → Rule Group → (referenced by) → Web ACL definition
                   NFW:  Rule → Rule Group → Firewall Policy
       ↓ FMS pushes / reconciles ↓
 Member Account (in scope)
   ├── FMS-created Web ACL (WAF)  ──associated──▶  ALB / API GW / CloudFront
   └── FMS-created Firewall + endpoints (NFW) ──routed──▶ VPC subnets
```

A more detailed WAF-centric relationship chain:

```
Rule ──contained in──▶ Rule Group ──referenced by──▶ Web ACL
Web ACL ──defined by──▶ FMS Security Policy (in delegated admin)
FMS Security Policy ──scoped to──▶ OUs / Accounts / Resource types / Tags
FMS ──creates in each member──▶ per-account Web ACL ──associated to──▶ protected resources
```

## 1.3 Where Does Everything Live?

| Resource | Lives in Security Infra (delegated admin) | Lives in member accounts | Created by whom |
|---|---|---|---|
| FMS Security Policy | Yes (the master copy) | No | You (Console/CLI/CloudFormation) |
| Custom WAF Rule Groups referenced by the policy | Yes (authored here) | Replicated/referenced per member as needed | You |
| AWS Managed Rule Groups | Owned by AWS, referenced everywhere | Referenced | AWS |
| IP Sets / Regex Pattern Sets used by your rule groups | Yes (with your rule groups) | Copied into FMS-managed per-account resources when needed | You |
| Web ACL (FMS-managed, name starts `FMManagedWebACLV2-`) | No | **Yes — one per member/region as needed** | **FMS, automatically** |
| Web ACL → ALB/API GW/CloudFront association | — | Yes | FMS, automatically |
| Network Firewall Firewall Policy (FMS-managed copy) | Authored centrally in the FMS policy | Deployed per member | FMS |
| Network Firewall endpoints + (optionally) subnets/route updates | No | Yes | FMS (depending on policy mode) |
| Logging destinations (Kinesis Firehose / S3 / CloudWatch Logs) | Often centralized in a logging account | Sometimes per-member | You |
| FMS compliance data / notification channel (SNS) | Yes | No | You / FMS |
| Organizations, OUs, accounts | Management account | — | You (management account) |

**What FMS creates automatically in member accounts:** per-account Web ACLs (WAF policies), Web ACL associations, remediation of drifted associations, Network Firewall firewalls/endpoints (and in "automatic" mode, firewall subnets and route table changes), Shield protections, security-group replicas/audits, and the service-linked role `AWSServiceRoleForFMS`.

**What remains centralized:** the FMS security policies themselves, your source rule groups/IP sets/regex sets, compliance dashboards, the FMS SNS notification channel, and (recommended) log aggregation.

**What exactly gets deployed into a member account (WAF example):** when an account with a tagged ALB falls in scope, FMS creates a Web ACL named like `FMManagedWebACLV2-<policyName>-<timestamp>` in that account/region, populates it with the rule groups defined in the policy (first/last rule groups around any account-local rules, per policy settings), associates it to the ALB, and keeps re-associating it if someone detaches it (when auto-remediation is enabled).

## 1.4 End-to-End Request Flow

**WAF path (HTTP request to an ALB in a member account):**

```
Client ──HTTP(S)──▶ ALB (member account)
                     │ 1. ALB hands request metadata to AWS WAF
                     ▼
           FMS-managed Web ACL (in member account)
                     │ 2. Rules evaluated in priority order:
                     │    - FMS "first" rule groups (from central policy)
                     │    - account-local rules (if policy allows)
                     │    - FMS "last" rule groups
                     │ 3. First terminating match wins:
                     │      Block  → 403 returned, request never reaches targets
                     │      Allow  → forwarded
                     │      Count/CAPTCHA/Challenge → per rule semantics
                     │ 4. No match → Web ACL default action (Allow/Block)
                     ▼
              ALB target group ──▶ application
                     │
                     └─ 5. WAF logs full request verdicts to
                        Firehose / S3 / CloudWatch Logs (if logging enabled)
```

**Network Firewall path (VPC traffic):**

```
Client / Internet ──▶ IGW ──route table──▶ Network Firewall endpoint (firewall subnet)
        1. Stateless rule groups evaluated first (5-tuple, very fast)
           → pass / drop / forward-to-stateful
        2. Stateful engine (Suricata rules, domain lists, 5-tuple stateful)
           → pass / drop / alert (order: strict or action-order per policy)
        3. Verdict applied; allowed packets follow VPC routes to workload subnet
        4. Flow + alert logs → S3 / CloudWatch Logs / Firehose
Workload subnet ◀──────────────────────────┘
```

The essential mental model: **you write intent once in the Security Infra account; FMS turns that intent into concrete WAF/NFW resources in 44 accounts and keeps them compliant.** Your CloudFormation migration therefore targets the *central* objects (FMS policies, your rule groups, IP sets, logging), not the per-account artifacts FMS generates.
# Part 2 – Existing Environment Discovery

Discovery is executed almost entirely from the **Security Infra account** (delegated admin), plus a few read-only calls in the **management account** for Organizations data. Run every CLI command **per region** where FMS/WAF is used; WAF `CLOUDFRONT` scope commands must run in **us-east-1**. Save all JSON output — it becomes the property values in your CloudFormation templates.

> Recommended: create a discovery folder per region, e.g. `discovery/eu-west-1/`, and redirect every command's output to a file. This snapshot is also your rollback reference.

## 2.1 Confirm the Delegated Administrator

**Console:** Management account → *AWS Organizations* → *Services* → *Firewall Manager* (shows delegated admin). Or Security Infra account → *AWS Firewall Manager* → *Settings*.

**CLI (management account):**

```bash
aws organizations list-delegated-administrators \
  --service-principal fms.amazonaws.com
aws fms get-admin-account            # run in Security Infra or mgmt account
```

*Why it matters:* every FMS API call and every FMS CloudFormation stack must run in this account. *Document:* the delegated admin account ID. *Needed in CFN:* determines the target account for all `AWS::FMS::Policy` stacks; never hard-code it in member templates.

## 2.2 Inventory FMS Security Policies

**Console:** Security Infra account → *AWS Firewall Manager* → *Security policies* → (per region) open each policy → note **Policy details, Scope (OUs/accounts, resource types, tags), Policy action (auto-remediate?), First/Last rule groups, Web ACL default action**.

**CLI:**

```bash
aws fms list-policies --max-items 100 > fms-list-policies.json
for id in $(jq -r '.PolicyList[].PolicyId' fms-list-policies.json); do
  aws fms get-policy --policy-id "$id" > "fms-policy-$id.json"
done
```

*Why:* the `get-policy` output is nearly a 1:1 source for `AWS::FMS::Policy` properties — especially `SecurityServicePolicyData.ManagedServiceData`, a JSON string containing the entire WAF/NFW configuration. *Document:* policy name, type (`WAFV2`, `NETWORK_FIREWALL`, `SECURITY_GROUPS_*`, `SHIELD_ADVANCED`, `DNS_FIREWALL`), scope (IncludeMap/ExcludeMap of OUs/accounts), `ResourceType(s)`, `ResourceTags`, `RemediationEnabled`, `DeleteUnusedFMManagedResources`. *Needed in CFN:* all of it, verbatim.

## 2.3 Web ACLs

**Console:** *WAF & Shield* → *Web ACLs* → select correct **region and scope (Regional / CloudFront)** → open each ACL → *Rules*, *Associated AWS resources*, *Logging and metrics*, *Custom response bodies*.

**CLI:**

```bash
aws wafv2 list-web-acls --scope REGIONAL > webacls-regional.json
aws wafv2 list-web-acls --scope CLOUDFRONT --region us-east-1 > webacls-cf.json
aws wafv2 get-web-acl --scope REGIONAL --name <NAME> --id <ID> > webacl-<NAME>.json
aws wafv2 list-resources-for-web-acl --web-acl-arn <ARN>      # associations (regional only)
```

*Why:* distinguishes (a) **FMS-managed** ACLs (`FMManagedWebACLV2-...` — do **not** migrate these; they're re-generated from the policy) from (b) **hand-made standalone** ACLs that must be modeled directly as `AWS::WAFv2::WebACL`. *Document:* name/ID/ARN, scope, default action, rule priorities and actions, WCU capacity, visibility config (CloudWatch metric names), associations. *Needed in CFN:* every property of standalone ACLs; rule order and `VisibilityConfig` are frequent template mistakes.

## 2.4 Rule Groups and Rules

**Console:** *WAF & Shield* → *Rule groups* (per region/scope) → open each → note capacity (WCU), rules, statements, actions, labels.

**CLI:**

```bash
aws wafv2 list-rule-groups --scope REGIONAL
aws wafv2 get-rule-group --scope REGIONAL --name <NAME> --id <ID> > rg-<NAME>.json
```

*Why:* custom rule groups are the reusable heart of your policies and must be recreated exactly — including **Capacity**, which is immutable after creation. *Document:* full `Rules[]` JSON, capacity, custom responses, labels. *Needed in CFN:* the entire rule group definition for `AWS::WAFv2::RuleGroup`; the ARN feeds the FMS policy's `ManagedServiceData`.

## 2.5 IP Sets and Regex Pattern Sets

**Console:** *WAF & Shield* → *IP sets* / *Regex pattern sets* (per region/scope).

**CLI:**

```bash
aws wafv2 list-ip-sets --scope REGIONAL
aws wafv2 get-ip-set --scope REGIONAL --name <NAME> --id <ID>
aws wafv2 list-regex-pattern-sets --scope REGIONAL
aws wafv2 get-regex-pattern-set --scope REGIONAL --name <NAME> --id <ID>
```

*Why:* rules reference these by ARN; if the ARN changes (recreate instead of import), every referencing rule must be updated. *Document:* name, IP version, addresses/patterns, and **which rules reference them**. *Needed in CFN:* `AWS::WAFv2::IPSet` / `AWS::WAFv2::RegexPatternSet` properties plus `!GetAtt ...Arn` wiring inside rule statements.

## 2.6 Logging Configuration

**Console:** each Web ACL → *Logging and metrics*; FMS console → policy → logging section; check Kinesis Data Firehose streams (must be named `aws-waf-logs-*`), S3 buckets, CloudWatch log groups. Also FMS → *Settings* → SNS notification channel.

**CLI:**

```bash
aws wafv2 list-logging-configurations --scope REGIONAL
aws wafv2 get-logging-configuration --resource-arn <WEB_ACL_ARN>
aws fms get-notification-channel
aws firehose list-delivery-streams; aws logs describe-log-groups \
  --log-group-name-prefix aws-waf-logs
```

*Why:* logging is the most commonly forgotten migration item; losing WAF logs breaks security monitoring silently. *Document:* destination ARNs, redacted fields, filters, cross-account delivery setup. *Needed in CFN:* `AWS::WAFv2::LoggingConfiguration` for standalone ACLs; the `loggingConfiguration` block inside FMS `ManagedServiceData` for policy-managed ACLs; `AWS::FMS::NotificationChannel`.

## 2.7 Resource Associations and Compliance

**CLI:**

```bash
aws wafv2 list-resources-for-web-acl --web-acl-arn <ARN> --resource-type APPLICATION_LOAD_BALANCER
aws fms list-compliance-status --policy-id <POLICY_ID>
aws fms get-compliance-detail --policy-id <POLICY_ID> --member-account <ACCOUNT_ID>
```

*Why:* your post-migration success criterion is "compliance state identical to pre-migration." *Document:* per-policy compliant/non-compliant account counts and any known violators (they are expected noise, not migration failures). *Needed in CFN:* nothing directly — this is your validation baseline.

## 2.8 Organizations, OUs, Accounts, Tags

**Console:** Management account → *AWS Organizations* → *AWS accounts* (tree view).

**CLI (management account or a delegated-read account):**

```bash
aws organizations list-roots
aws organizations list-organizational-units-for-parent --parent-id <ROOT_OR_OU_ID>
aws organizations list-accounts-for-parent --parent-id <OU_ID>
aws organizations list-accounts > all-accounts.json
aws organizations list-tags-for-resource --resource-id <ACCOUNT_OR_OU_ID>
```

*Why:* FMS policy scope references OU IDs (`ou-xxxx-xxxxxxxx`) and account IDs; a typo silently changes which of the 44 accounts are protected. *Document:* the full OU tree with IDs, and which OUs each policy includes/excludes. *Needed in CFN:* the `IncludeMap`/`ExcludeMap` values of `AWS::FMS::Policy`, and StackSet deployment targets.

## 2.9 Network Firewall (if used)

**Console:** *VPC* → *Network Firewall* → *Firewalls / Firewall policies / Rule groups* in the delegated admin (central definitions) and spot-check member accounts (FMS-created firewalls).

**CLI:**

```bash
aws network-firewall list-firewall-policies
aws network-firewall describe-firewall-policy --firewall-policy-name <NAME>
aws network-firewall list-rule-groups
aws network-firewall describe-rule-group --rule-group-name <NAME> --type STATEFUL
aws network-firewall list-firewalls
aws network-firewall describe-firewall --firewall-name <NAME>
aws network-firewall describe-logging-configuration --firewall-name <NAME>
```

*Why:* FMS Network Firewall policies embed the firewall policy definition inside `ManagedServiceData`, including endpoint placement mode (automatic vs. custom subnets) and route management. *Document:* rule group ARNs/capacities, stateless default actions, stateful rule order, endpoint/AZ layout, logging destinations. *Needed in CFN:* `AWS::NetworkFirewall::RuleGroup` definitions plus the NFW section of the FMS policy's `ManagedServiceData`.

## 2.10 Discovery Output Checklist

By the end of Part 2 you should have, per region: `fms-policy-*.json` for every policy; `get-rule-group`/`get-ip-set`/`get-regex-pattern-set` JSON for every custom object; logging configuration JSON; the OU tree with IDs; a compliance baseline export; and a written classification of every Web ACL as **FMS-managed (skip)** or **standalone (migrate)**. This bundle *is* your Low-Level Design.
# Part 3 – Migration Planning

An enterprise-grade migration runs in nine phases. Each phase exists to remove a specific class of production risk.

**Phase 1 – Discovery (Part 2).** *Why:* you cannot safely codify what you have not measured; undiscovered dependencies (an IP set referenced by two rule groups, a forgotten CloudFront-scope ACL in us-east-1) are the #1 cause of migration outages.

**Phase 2 – Documentation.** Convert raw JSON into a human-readable inventory: one sheet per policy with scope, rule groups, logging, associations, and owner. *Why:* reviewers, approvers, and the rollback operator cannot work from raw API dumps; documentation is also the artifact your change board will approve.

**Phase 3 – Backup.** Store all discovery JSON in versioned S3 (and Git), export FMS policy JSON via `get-policy`, and snapshot compliance state. *Why:* if anything goes wrong, the fastest rollback is "re-apply the exported JSON with `aws fms put-policy`" — that only works if the export exists and is trusted.

**Phase 4 – Risk assessment.** For each resource decide: import (zero-touch), shadow-deploy (create parallel then cut over), or recreate (accepted downtime). Classify policies by blast radius (a WAF policy scoped to the Prod OU is critical; a Count-mode pilot policy is low risk). Identify the failure modes: FMS deleting and recreating member Web ACLs, WCU capacity errors, logging gaps, association flaps. *Why:* it converts "migration" from one big risk into many small, individually mitigated risks and drives the deployment order.

**Phase 5 – Change management.** Raise a change record per policy (not one giant change), with maintenance window, communication plan to the 44 account owners, approval from security engineering, and a named rollback operator. *Why:* FMS changes affect every in-scope account simultaneously; stakeholders must know a config plane change is occurring even when zero traffic impact is expected.

**Phase 6 – Rollback planning.** Written, rehearsed procedures: (a) CloudFormation stack rollback / `cancel-update-stack`; (b) for imports gone wrong, `DeletionPolicy: Retain` + stack deletion leaves the live resource untouched; (c) worst case, `aws fms put-policy --cli-input-json file://backup.json` restores the pre-migration policy. *Why:* rollback designed during an incident is guesswork; rollback designed before is a procedure.

**Phase 7 – Validation (non-prod first).** Deploy the full template set into a sandbox OU/test policy, compare `get-policy` output old-vs-new with a JSON diff, verify FMS creates the member Web ACL and associations, send test traffic that must be blocked/allowed, confirm logs arrive. *Why:* `ManagedServiceData` is a JSON-in-JSON string — the only reliable test of correctness is a live FMS reconciliation.

**Phase 8 – Production deployment (Part 6).** Import-first, lowest-risk-policy-first, one policy per change window, with change sets reviewed before execution.

**Phase 9 – Post-deployment verification.** Re-run the compliance export and diff against the Phase 3 baseline; run drift detection; verify logging volume unchanged; keep hypercare monitoring for 1–2 weeks. *Why:* FMS reconciliation is asynchronous — a policy can look fine at deploy time and churn member resources hours later.

---

# Part 4 – CloudFormation Design

## 4.1 Resource-by-Resource Mapping

### AWS::WAFv2::IPSet / AWS::WAFv2::RegexPatternSet

- **Dependency order:** first — everything else references their ARNs.
- **Required:** `Name` is optional but set it (avoids ugly generated names); `Scope` (`REGIONAL`/`CLOUDFRONT`), `Addresses`/`RegularExpressionList`, `IPAddressVersion` (IPSet).
- **Optional:** `Description`, `Tags`.
- **Import support:** **Yes** (`aws cloudformation` resource import). Import preserves the ARN, so existing rule references keep working — strongly preferred.
- **Common mistakes:** wrong scope region (CLOUDFRONT stacks must be deployed in us-east-1); CIDRs not normalized (`1.2.3.4/32` required, not `1.2.3.4`).

### AWS::WAFv2::RuleGroup

- **Dependency order:** after IP/Regex sets, before Web ACLs and FMS policies.
- **Required:** `Capacity` (WCU — **immutable**; copy the discovered value, don't guess), `Scope`, `VisibilityConfig` (also required per rule).
- **Optional:** `Rules`, `CustomResponseBodies`, `Tags`, `Description`.
- **Import support:** **Yes.** Import keeps the ARN so FMS policies referencing it need no change.
- **Common mistakes:** under-sizing `Capacity` (update then fails forever — recreation required); forgetting per-rule `VisibilityConfig`; duplicated rule `Priority` values; recreating instead of importing and thereby changing the ARN referenced inside FMS `ManagedServiceData`.

### AWS::WAFv2::WebACL (standalone ACLs only)

- **Dependency order:** after rule groups/IP sets.
- **Required:** `DefaultAction`, `Scope`, `VisibilityConfig`.
- **Optional:** `Rules`, `CustomResponseBodies`, `CaptchaConfig`, `ChallengeConfig`, `AssociationConfig`, `TokenDomains`, `Tags`.
- **Import support:** **Yes**, and `AWS::WAFv2::WebACLAssociation` also exists for regional associations (CloudFront associates via the distribution's `WebACLId`).
- **Common mistakes:** attempting to import or recreate **FMS-managed** ACLs (never do this — FMS owns them and will fight you); managed rule group `OverrideAction` vs rule `Action` confusion (`OverrideAction: {None: {}}` to honor group actions, `{Count: {}}` to monitor).

### AWS::FMS::Policy

- **Dependency order:** last — after the rule groups whose ARNs it references. Deployed **only in the Security Infra account**.
- **Required:** `PolicyName`, `SecurityServicePolicyData` (`Type` + `ManagedServiceData` JSON string), `ExcludeResourceTags`, `RemediationEnabled`, `ResourceType` (or `ResourceTypeList` via `ResourceSetIds`/type list depending on policy type).
- **Optional:** `IncludeMap`/`ExcludeMap` (`ORGUNIT`/`ACCOUNT` lists), `ResourceTags`, `ResourcesCleanUp`, `DeleteAllPolicyResources` behavior on delete, `PolicyDescription`, `Tags`.
- **Import support:** **Yes** — `AWS::FMS::Policy` is an importable registry resource; importing the existing policy (by `PolicyId`) is the zero-risk path because FMS never sees a delete/recreate.
- **Common mistakes:** hand-writing `ManagedServiceData` instead of copying it from `get-policy` (it is strict JSON with type-specific schema); recreating a policy instead of importing it — **deleting an FMS policy can delete every FMS-managed Web ACL in all 44 accounts**, momentarily removing protection; forgetting that updating the *template* JSON string with re-ordered keys still counts as an update (normalize the JSON once and keep it stable).

### AWS::FMS::NotificationChannel and AWS::FMS::ResourceSet

- Model the SNS channel (`SnsTopicArn`, `SnsRoleName`) and any resource sets used for scoping. Both support import. Common mistake: SNS role missing `fms.amazonaws.com` trust.

### AWS::NetworkFirewall::RuleGroup / ::FirewallPolicy / ::Firewall (if applicable)

- For FMS-managed NFW, you usually only codify **rule groups** centrally (their ARNs go into the FMS policy's `ManagedServiceData`); FMS creates the per-account firewall policy and firewalls. Standalone (non-FMS) firewalls map to all three types.
- `RuleGroup` requires `Capacity` (immutable), `Type` (`STATELESS`/`STATEFUL`), `RuleGroupName`. `Firewall` requires VPC/subnet mappings.
- **Import support:** yes for these types (verify with `aws cloudformation list-types`/docs in your region before relying on it).
- **Common mistakes:** stateful rule order (`STRICT_ORDER` vs default) mismatch between discovered policy and template; capacity guessing.

## 4.2 Recreate vs. Import Decision Table

| Resource | Preferred path | Why |
|---|---|---|
| FMS Policy | **Import** existing PolicyId | Delete/recreate can strip protection org-wide |
| Custom Rule Groups, IP Sets, Regex Sets | **Import** | Preserves ARNs referenced by policies |
| Standalone Web ACLs + associations | Import | Avoids a protection gap on the attached resource |
| FMS-managed Web ACLs (`FMManagedWebACLV2-*`) | **Do not touch** | Owned by FMS; regenerated from policy |
| Logging configs | Recreate via template after import of parent | Import support is limited; recreation is low risk |
| NFW rule groups (FMS-referenced) | Import | ARN preservation |

## 4.3 Repository and Stack Layout

```
fms-iac/
├── README.md
├── docs/                      # inventory sheets, runbook, diagrams
├── discovery/<region>/        # raw JSON exports (source of truth snapshot)
├── templates/
│   ├── 00-prereqs/logging.yaml          # Firehose aws-waf-logs-*, S3, IAM
│   ├── 10-waf-shared/ip-sets.yaml
│   ├── 10-waf-shared/regex-sets.yaml
│   ├── 20-waf-rulegroups/rg-<name>.yaml
│   ├── 25-nfw-rulegroups/               # if applicable
│   ├── 30-webacls/standalone-<name>.yaml
│   ├── 40-fms-policies/policy-<name>.yaml
│   └── 50-fms-settings/notification-channel.yaml
├── params/{dev,prod}/*.json
├── import/                    # resources-to-import mapping files
└── pipeline/                  # CI/CD definitions
```

**Modular stack design rules:** one stack per lifecycle boundary (shared sets ≠ rule groups ≠ policies); export rule-group ARNs via `Outputs`/`Fn::ImportValue` or SSM parameters; separate stacks per region; per-policy FMS stacks so one change set touches one policy; `DeletionPolicy: Retain` on every imported security resource.
# Part 5 – UI Walkthrough

## 5.1 Delegated Admin (Security Infra Account)

```
AWS Console (Security Infra, correct region)
→ AWS Firewall Manager
  → Security policies
    → [select policy]
      → Policy details tab      : type, remediation, resource cleanup
      → Scope                   : OUs/accounts (IncludeMap/ExcludeMap),
                                  Resource type(s), Resource tags
      → Policy rules            : first/last rule groups, default action,
                                  logging configuration
      → Accounts and resources  : per-account COMPLIANT / NON-COMPLIANT
        → [select member account] → violating resources & reasons
```

Also visit: *Firewall Manager → Settings* (SNS channel), *WAF & Shield → Rule groups / IP sets / Regex pattern sets* (your central building blocks), and for NFW: *VPC → Network Firewall → Rule groups*.

## 5.2 Tracing a Policy into a Member Account — AWS WAF

Log in to the member account (same region as the protected resource; **us-east-1 + Global scope** for CloudFront):

```
Member Account Console
1. Policy deployment  : WAF & Shield → Web ACLs → find FMManagedWebACLV2-<policy>…
2. Web ACL contents   : open it → Rules tab → confirm the policy's first/last
                        rule groups appear with expected priorities
3. Associations       : same ACL → Associated AWS resources tab
                        → the protected ALB / API Gateway stage / AppSync etc.
4. Protected resource : EC2 → Load Balancers → [ALB] → Integrations/Attributes
                        shows attached Web ACL; API Gateway → stage → Web ACL;
                        CloudFront → distribution → Security → AWS WAF
5. Logging            : Web ACL → Logging and metrics → destination
                        (Firehose/S3/CW Logs) + CloudWatch → Metrics → WAFV2
6. Compliance (back in Security Infra): FMS → policy → Accounts and resources
                        → the member shows COMPLIANT
```

## 5.3 Tracing a Policy into a Member Account — AWS Network Firewall

```
Member Account Console
1. Policy deployment  : VPC → Network Firewall → Firewalls
                        → FMS-created firewall (FMManagedNetworkFirewall…)
2. Firewall policy    : open firewall → Firewall policy → stateless/stateful
                        rule groups pushed from the central FMS policy
3. Endpoints          : firewall → Firewall endpoints → one per AZ,
                        in the firewall subnets
4. Routing (the association) : VPC → Route tables → IGW/subnet route tables
                        send traffic through vpce-… firewall endpoints
5. Protected VPC      : confirm workload subnets route via the endpoint
6. Logging            : firewall → Logging → flow/alert destinations
7. Compliance         : back in Security Infra FMS console, as above
```

**Key WAF vs NFW difference:** WAF's "association" is a direct ACL→resource attachment visible on the resource; NFW's "association" is *routing* — verification means reading route tables, not an attachment field. NFW also involves subnets/AZs, so FMS policy mode (automatic endpoint placement vs. custom) changes what you should expect to see.

---

# Part 6 – CloudFormation Deployment

End-to-end sequence — run in the **Security Infra account**, per region:

**1. Discovery** — complete Part 2; freeze a snapshot tag (e.g., `pre-iac-2026-07`). *Order rationale:* templates are transcriptions of this data; stale discovery = wrong templates.

**2. Export** — for each object save the exact JSON (`get-policy`, `get-rule-group`, …) into `discovery/`. This JSON, minus read-only fields, is pasted into templates.

**3. Documentation** — inventory sheets + this runbook, approved by change management.

**4. Template creation** — author templates in dependency order (IP/Regex sets → rule groups → standalone ACLs → FMS policies → notification channel). Copy `ManagedServiceData` verbatim; add `DeletionPolicy: Retain` to imported resources.

**5. Validation** —

```bash
aws cloudformation validate-template --template-body file://rg-core.yaml
cfn-lint templates/**/*.yaml
```

plus peer review diffing template properties against exported JSON.

**6. Non-production test** — deploy everything against a **test FMS policy scoped to a sandbox OU** (or Count-mode rules). Verify FMS creates the member ACL, associations, logs, and that test traffic is handled as expected. *Rationale:* the only true validator of `ManagedServiceData` is FMS itself.

**7. Change set creation** — production path is **import first**:

```bash
aws cloudformation create-change-set \
  --stack-name fms-policy-prod-waf \
  --change-set-name import-existing \
  --change-set-type IMPORT \
  --resources-to-import file://import/policy-prod-waf.json \
  --template-body file://templates/40-fms-policies/policy-prod-waf.yaml \
  --capabilities CAPABILITY_NAMED_IAM
aws cloudformation describe-change-set --change-set-name import-existing \
  --stack-name fms-policy-prod-waf     # review: must show ONLY Import actions
```

*Rationale:* the change set is your last look before anything touches production; for imports it must show zero Modify/Replace actions.

**8. Production deployment** — `execute-change-set`, one policy per change window, lowest blast radius first (Count-mode/pilot policies → non-prod OU policies → prod OU policies). Immediately run **drift detection** on the imported stack; resolve any drift by correcting the template, not the resource.

**9. Verification** — re-run compliance export and JSON-diff vs baseline; `aws fms get-policy` diff vs pre-migration export (only `PolicyUpdateToken` should differ); spot-check one member account per OU via Part 5; confirm WAF log delivery volume in CloudWatch.

**10. Rollback (if required)** — in order of preference: (a) `cancel-update-stack` / automatic rollback for failed updates; (b) delete the stack — `Retain` policies leave live resources untouched — and continue operating manually; (c) restore config with `aws fms put-policy --cli-input-json file://discovery/<region>/fms-policy-<id>.json` (strip read-only fields). Never "roll back" by deleting an FMS policy.

**Why order matters:** ARN producers must exist before consumers (sets → rule groups → policies); imports must precede any updates (you can't update what CloudFormation doesn't own); low-blast-radius before high gives you a rehearsal on every riskier step; and validation gates between steps ensure that at any failure point the live environment is still the untouched original.
# Part 7 – Best Practices

**Version control.** All templates, parameter files, and the discovery snapshot live in Git. Protected main branch, PR review by a second security engineer, tags per production release. The Git history becomes your audit trail of every WAF rule change.

**CI/CD.** Pipeline stages: `cfn-lint` + `validate-template` → security static analysis (cfn-nag/Checkov) → deploy to sandbox policy → automated smoke test (send known-bad request, expect 403) → manual approval → create change set in prod → manual review of change set → execute. Humans approve; pipelines execute — no console changes after migration.

**StackSets.** FMS itself distributes protections, so you usually do *not* StackSet the FMS policy (it lives only in the delegated admin). Use **service-managed StackSets** from the delegated admin for anything genuinely per-account (e.g., member-side logging IAM, standalone ACLs owned by app teams), targeting OUs so new accounts auto-provision.

**Drift detection.** Schedule `detect-stack-drift` (e.g., weekly via EventBridge + Lambda) on all FMS/WAF stacks; alert on any non-`IN_SYNC` result. Pair with an SCP denying `wafv2:*`/`fms:*` writes outside the pipeline role so drift becomes rare by construction.

**Change sets.** Mandatory for every production update; direct `update-stack` is disabled by pipeline policy. Reviewers check for `Replacement: True` on any WAF resource — replacement of a rule group changes its ARN and can momentarily degrade protection.

**CloudTrail.** Organization trail capturing `fms.amazonaws.com`, `wafv2.amazonaws.com`, `network-firewall.amazonaws.com` in all accounts; alerts on out-of-band `PutPolicy`, `UpdateWebACL`, `DisassociateWebACL`.

**AWS Config.** Enable org-wide with rules such as `fms-webacl-resource-policy-check`, `alb-waf-enabled`, `api-gw-associated-with-waf`; aggregate into the Security Infra account. Config answers "is every ALB actually protected?" independently of FMS's own compliance view.

**Monitoring & logging.** CloudWatch dashboards per Web ACL (Allowed/Blocked/Counted), anomaly alarms on BlockedRequests spikes and on WAF log delivery volume dropping to zero (silent logging failure). Centralize WAF/NFW logs into the log-archive account (Firehose → S3, Athena table for investigation).

**Security & least-privilege IAM.** Separate pipeline roles: `fms-policy-deployer` (fms:*, in delegated admin only), `waf-rulegroup-deployer` (wafv2 on tagged resources), read-only discovery role. No human write access in prod. Deny deletion of the logging bucket. Protect the delegated-admin registration itself (only the management account can change it).

**Cross-account deployment.** Pipeline (e.g., in a shared-services account) assumes a deployer role in the Security Infra account via `sts:AssumeRole` with `ExternalId`; StackSets handle member-account needs. Never distribute static keys.

**Multi-region.** WAF `CLOUDFRONT` scope and its stacks: us-east-1 only. Regional policies: replicate stacks per region with identical templates + per-region parameter files. FMS policies are regional — a "global" WAF standard means one policy per active region; keep them in sync via the same template.

**Disaster recovery.** The Git repo + discovery snapshots are the DR plan: from an empty delegated admin you can redeploy every policy in dependency order. Rehearse annually in a sandbox org. Back up: templates (Git), policy JSON (S3, versioned, object-lock), and the runbook itself. RTO driver is FMS reconciliation time (~minutes to tens of minutes across 44 accounts), not template deploy time.

---

# Part 8 – Visual Diagrams

## 8.1 Overall Architecture

```
                          AWS ORGANIZATION
┌─────────────────────────────────────────────────────────────────┐
│  Management Account                                             │
│   • Organizations, OUs, accounts                                │
│   • Delegates FMS admin ──────────────┐                         │
└───────────────────────────────────────┼─────────────────────────┘
                                        ▼
┌─────────────────────────────────────────────────────────────────┐
│  SECURITY INFRA ACCOUNT (FMS Delegated Administrator)           │
│   CloudFormation stacks:                                        │
│    [IP/Regex Sets] → [Rule Groups] → [FMS Policies] → [SNS]     │
│   FMS console: compliance across 44 accounts                    │
└───────────────┬───────────────────────────┬─────────────────────┘
        FMS reconciles                FMS reconciles
                ▼                           ▼
┌──────────────────────────┐   ┌──────────────────────────┐
│ Member Account A (Prod)  │   │ Member Account B (NonProd)│  × ~44
│  FMManagedWebACLV2-…     │   │  FMManagedWebACLV2-…      │
│    └─▶ ALB / API GW      │   │    └─▶ ALB                │
│  NFW endpoints ─▶ VPC    │   │  WAF logs ─▶ Firehose     │
└──────────────────────────┘   └──────────────────────────┘
```

## 8.2 Firewall Manager Deployment Flow

```
Engineer commits template ─▶ CI/CD ─▶ Change Set ─▶ Execute
        │                                             │
        ▼                                             ▼
  Git (source of truth)                    AWS::FMS::Policy updated
                                                      │
                                     FMS evaluates scope (OUs/tags/types)
                                                      │
              ┌───────────────────────────────────────┼──────────────┐
              ▼                                       ▼              ▼
      Account in scope?                      Resource in scope?   New account
       yes → ensure Web ACL exists            yes → associate      joins OU →
       + matches policy definition            no  → (cleanup*)     auto-protected
              │
              ▼
   Compliance status reported back to Security Infra account
   (* if resource cleanup is enabled on the policy)
```

## 8.3 WAF Object Chain

```
Rule ─▶ Rule Group ─▶ Web ACL (definition inside FMS policy)
                          │
              FMS Security Policy (delegated admin)
                          │  scope: OU / account / tags / resource type
                          ▼
                 Member Account (per region)
                          │  FMS creates FMManagedWebACLV2-…
                          ▼
              Protected Resource (ALB / API GW / CloudFront)
                          │
                          ▼
             Request evaluated ─▶ Allow / Block / Count
```

## 8.4 CloudFormation Dependency Graph

```
AWS::WAFv2::IPSet ┐
AWS::WAFv2::RegexPatternSet ┴─▶ AWS::WAFv2::RuleGroup ─▶ AWS::FMS::Policy
                                        │                       ▲
                                        └─▶ AWS::WAFv2::WebACL  │
                                             (standalone)       │
AWS::NetworkFirewall::RuleGroup ────────────────────────────────┘
Logging stack (Firehose/S3/IAM)  ─▶ referenced by policies/ACLs
AWS::FMS::NotificationChannel    ─▶ independent, deploy last
(deploy left → right; delete right → left)
```

## 8.5 Resource Relationships (who owns what)

```
 YOU own (in CloudFormation, Security Infra acct):
   FMS Policy ── references ──▶ RuleGroup ── references ──▶ IPSet/RegexSet
        │                                            
 FMS owns (never import):                            
        └─ creates ─▶ FMManagedWebACLV2-* ── associated ─▶ ALB (member)
        └─ creates ─▶ Member NFW firewall ── routed ─▶ VPC subnets
 AWS owns: Managed Rule Groups (referenced by name/vendor)
```

## 8.6 Request Processing Flow

```
        ┌────────┐   HTTPS   ┌─────┐  inspect  ┌──────────────────┐
Client ─▶│CloudFront│─ or ──▶│ ALB │──────────▶│ Web ACL (WAF)    │
        └────────┘           └─────┘           │ 1 first RGs (FMS)│
                                               │ 2 local rules    │
                                               │ 3 last RGs (FMS) │
                                               │ 4 default action │
                                               └───┬─────────┬────┘
                                              Block│         │Allow
                                                   ▼         ▼
                                              HTTP 403   Target app
                                                   └────┬────┘
                                                        ▼
                                            WAF logs ─▶ Firehose ─▶ S3/SIEM
```

---

# Appendix – Quick Reference

| Task | Command |
|---|---|
| Confirm delegated admin | `aws fms get-admin-account` |
| Export a policy | `aws fms get-policy --policy-id <id>` |
| List member compliance | `aws fms list-compliance-status --policy-id <id>` |
| Export rule group | `aws wafv2 get-rule-group --scope REGIONAL --name <n> --id <id>` |
| Import change set | `aws cloudformation create-change-set --change-set-type IMPORT …` |
| Drift check | `aws cloudformation detect-stack-drift --stack-name <s>` |
| Emergency restore | `aws fms put-policy --cli-input-json file://backup.json` |

**Final reminders:** never import FMS-managed member resources; never delete an FMS policy as a rollback; `Capacity` (WCU) is immutable; CloudFront scope lives in us-east-1; the exported `get-policy` JSON is your template's source of truth.
