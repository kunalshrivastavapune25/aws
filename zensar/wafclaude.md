# AWS Firewall Manager → CloudFormation Migration Guide
### End-to-end runbook for a 44-account AWS Organization

---

## PART 1 — Conceptual Architecture

### 1.1 The building blocks, explained simply

**AWS Organizations** — the container that groups all 44 accounts under one root. It lets you apply policies (SCPs) and delegate admin rights for services like Firewall Manager centrally instead of doing it account-by-account.

**Management account** — the "root" account of the Organization. It's the only account that can enable Firewall Manager as a service and designate a delegated administrator. It should otherwise do as little day-to-day work as possible — it's a control-plane account, not an operational one.

**Delegated administrator** — a member account (your **Security Infra account**) that the management account grants permission to administer Firewall Manager on the Organization's behalf. This is the account where you actually build policies, view compliance, and manage rule groups — without needing to log into the management account for routine work.

**AWS Firewall Manager (FMS)** — an orchestration layer. It does not enforce traffic itself. It creates and pushes WAF Web ACLs (or Network Firewall policies, security groups, etc.) into member accounts according to rules you define once, centrally. Think of it as a policy engine that "reaches into" every account/OU you scope it to and keeps a resource in a compliant state.

**AWS WAF** — the actual traffic-inspection engine for HTTP(S). It evaluates a **Web ACL** against **Rules** (or **Rule Groups**, which are just reusable bundles of rules) and returns ALLOW / BLOCK / COUNT for each request. WAF attaches to CloudFront distributions, ALBs, API Gateway REST APIs, AppSync, Cognito user pools, or Verified Access instances.

**AWS Network Firewall** (if in scope) — a VPC-level, stateful/stateless network traffic inspection service that sits in a VPC's routing path. It uses **Firewall Policies** built from **Rule Groups** (5-tuple, domain-list, or Suricata-compatible rules) to allow/deny/alert on packets before they reach your workloads. Distinct from WAF: Network Firewall inspects packets at L3/L4 (and L7 for TLS SNI/domain filtering); WAF inspects HTTP(S) requests at L7.

**Rule** — a single condition + action (e.g., "block if SQLi pattern matched").

**Rule Group** — a named, reusable collection of Rules. Either "managed" (AWS or Marketplace-provided, e.g., AWSManagedRulesCommonRuleSet) or "customer-owned" (you write the rules).

**Web ACL** — the top-level WAF object. Lists Rule Groups/Rules in priority order with a default action (Allow or Block). This is what actually attaches to a protected resource.

**Firewall Policy** — the Network Firewall equivalent of a Web ACL: an ordered set of stateless and stateful Rule Groups plus default actions, applied to VPC subnets via firewall endpoints.

**Firewall Manager Security Policy** — the central object you author once in the delegated admin account. It says, in effect: "for every account/resource matching this OU/tag scope, ensure a Web ACL (or Network Firewall policy, or Security Group config) exists with this Rule Group configuration, and auto-remediate if it drifts." This is the object that gets pushed out to accounts.

**Resource Association** — the linkage between a protected resource (ALB, CloudFront distribution, VPC, etc.) and the Web ACL / Firewall Policy that Firewall Manager created for it. For WAF policies scoped by resource type, FMS creates the association automatically. For Network Firewall, FMS creates firewall endpoints and updates route tables in each in-scope VPC.

**Organizational Units (OUs)** — folders inside AWS Organizations used to group accounts (e.g., `Production`, `Non-Production`, `Sandbox`). Firewall Manager policies are most commonly scoped to OUs rather than individual accounts, so new accounts placed in an OU automatically inherit protection.

**Member accounts** — the 44 accounts in scope. They do not need to take any action — FMS deploys into them using the Organizations trust relationship (no per-account IAM role setup required, unlike some other cross-account patterns you've used with StackSets).

### 1.2 How it all relates — flow chart

```
                         AWS ORGANIZATION (Management Account)
                                      │
                          enables FMS + delegates admin
                                      │
                                      ▼
                     SECURITY INFRA ACCOUNT (Delegated Admin)
                     ┌─────────────────────────────────────┐
                     │  Rule Groups  (customer + managed)   │
                     │       │                               │
                     │       ▼                               │
                     │  Web ACL / Firewall Policy (template)│
                     │       │                               │
                     │       ▼                               │
                     │  Firewall Manager Security Policy    │
                     │       (scope: OU / tags / account IDs)│
                     └───────────────┬───────────────────────┘
                                     │  FMS pushes policy
                 ┌───────────────────┼───────────────────┐
                 ▼                   ▼                   ▼
         MEMBER ACCOUNT A     MEMBER ACCOUNT B    ...   MEMBER ACCOUNT 44
         (auto-created)       (auto-created)             (auto-created)
         ┌───────────────┐    ┌───────────────┐
         │ FMS-managed   │    │ FMS-managed   │
         │ Web ACL       │    │ Web ACL       │
         │   │           │    │   │           │
         │   ▼           │    │   ▼           │
         │ Associated to │    │ Associated to │
         │ ALB/CF/APIGW  │    │ ALB/CF/APIGW  │
         └───────────────┘    └───────────────┘
```

### 1.3 Where things live

| Resource | Lives in Security Infra (delegated admin) | Distributed into member accounts | Created automatically by FMS |
|---|---|---|---|
| Customer Rule Groups referenced by policy | ✅ (authored here) | Copy pushed as read-only reference | ✅ |
| FMS Security Policy | ✅ (only place it's authored) | — | n/a |
| Web ACL (per protected resource) | Only if the resource itself is in the admin account | ✅ one per account/region in scope | ✅ |
| Resource association | — | ✅ | ✅ |
| Network Firewall Firewall Policy | ✅ authored | Instantiated copy in each VPC's account | ✅ |
| Network Firewall endpoints + route table updates | — | ✅ | ✅ |
| Compliance data | ✅ viewed centrally | — (read-only rollup) | n/a |

**Key architectural fact for your CFN migration:** you do **not** need cross-account StackSets to push FMS policy the way you did for `DnsVpcAttachmentRole`. FMS's own Organizations integration handles the fan-out. Your CloudFormation only needs to manage objects **in the Security Infra account** (Rule Groups, Web ACL template, FMS Policy resource). The per-member-account Web ACLs are FMS-managed and are **not** things you should try to import into CloudFormation individually — more on this in Part 4.

### 1.4 Request flow (HTTP request through the stack)

```
Client
  │
  ▼
CloudFront / ALB / API Gateway  (protected resource, in a member account)
  │
  ▼
Associated Web ACL  (FMS-managed, lives alongside the resource)
  │
  ▼
Rule Group evaluation (priority order)
  ├─ Managed Rule Group (e.g., Core Rule Set) ── first, usually
  ├─ Customer Rule Group (org-specific rules)
  └─ Rate-based rules
  │
  ▼
Default action: ALLOW or BLOCK
  │
  ├─ BLOCKED → 403 returned to client, logged to WAF logs (Kinesis Firehose → S3/CloudWatch)
  └─ ALLOWED → request proceeds to origin/backend
```

If Network Firewall is also in scope, replace the top of the chain with: `Client → Internet/TGW → VPC subnet with Firewall Manager-provisioned Firewall Endpoint → stateless rules → stateful rules → allow/deny → route continues to workload subnet`.

---

## PART 2 — Existing Environment Discovery

Do this from the **Security Infra (delegated admin) account**, using a role with `FMS:Get*`, `FMS:List*`, `wafv2:Get*`, `wafv2:List*`, `network-firewall:Describe*`, `organizations:List*`, `organizations:Describe*` at minimum.

### 2.1 Firewall Manager delegated administrator

**Console:** Firewall Manager → Settings → confirms the current delegated admin account ID and enabled regions.

**CLI:**
```bash
aws fms get-admin-account
aws organizations list-delegated-administrators --service-principal fms.amazonaws.com
```
*Why it matters:* your CFN deployment principal must run from this exact account — FMS Policy resources can only be created here.
*Document:* account ID, enabled Organizations service access status, list of enabled regions for FMS.

### 2.2 Security Policies

**Console:** Firewall Manager → Security policies → note each policy name, type (WAF, Network Firewall, Security Group, etc.).

**CLI:**
```bash
aws fms list-policies --max-results 100
aws fms get-policy --policy-id <policy-id>
```
*Document per policy:* PolicyId, PolicyName, ResourceType (e.g., `AWS::ElasticLoadBalancingV2::LoadBalancer`, `AWS::CloudFront::Distribution`), SecurityServicePolicyData (this JSON blob is your Web ACL template — capture verbatim), ExcludeResourceTags flag, ResourceTags, IncludeMap/ExcludeMap (OU or account scoping — **this is your CFN `IncludeMap`/`ExcludeMap` property**), RemediationEnabled, DeleteUnusedFMManagedResources.
*Needed in CFN:* every field above maps almost 1:1 to `AWS::FMS::Policy` properties — this is the single most important discovery artifact.

### 2.3 Web ACLs (regional + CloudFront/global)

**Console:** WAF & Shield → Web ACLs (toggle Region dropdown; CloudFront ACLs only show under "Global (CloudFront)").

**CLI:**
```bash
aws wafv2 list-web-acls --scope REGIONAL --region <region>
aws wafv2 list-web-acls --scope CLOUDFRONT --region us-east-1
aws wafv2 get-web-acl --name <name> --scope REGIONAL --id <id> --region <region>
```
*Document:* Name, Id, ARN, DefaultAction, Rules array (name, priority, statement, action, visibility config), CustomResponseBodies, CAPTCHA/Challenge config.
*Note:* FMS-managed Web ACLs are named with an `FMManagedWebACLV2-<policyname>-<id>` pattern — flag these as "do not manage directly," since FMS owns their lifecycle. Only the **template** (defined inside the FMS policy) needs to become IaC.

### 2.4 Rule Groups

**CLI:**
```bash
aws wafv2 list-rule-groups --scope REGIONAL --region <region>
aws wafv2 get-rule-group --name <name> --scope REGIONAL --id <id> --region <region>
aws wafv2 list-available-managed-rule-groups --scope REGIONAL --region <region>
```
*Document:* every customer-owned Rule Group's full Rules JSON (statements, byte-match, regex, geo-match, rate-based configs, label match), Capacity (WCU — required at creation time and cannot be increased later without recreation), VisibilityConfig.
*Needed in CFN:* becomes `AWS::WAFv2::RuleGroup` resources, one per group. Capture Capacity exactly — CFN requires it up front.

### 2.5 Rules (inline, if any exist outside rule groups)

Captured as part of `get-web-acl` / `get-rule-group` output above — WAFv2 rules are always nested JSON inside a Web ACL or Rule Group, there's no separate `list-rules` call. Document inline rules on the Web ACL itself separately from rule-group-sourced rules, since inline rules become part of the `Rules` property of `AWS::WAFv2::WebACL` directly.

### 2.6 IP Sets and Regex Pattern Sets

**CLI:**
```bash
aws wafv2 list-ip-sets --scope REGIONAL --region <region>
aws wafv2 get-ip-set --name <name> --scope REGIONAL --id <id> --region <region>
aws wafv2 list-regex-pattern-sets --scope REGIONAL --region <region>
aws wafv2 get-regex-pattern-set --name <name> --scope REGIONAL --id <id> --region <region>
```
*Document:* every IP address/CIDR in each set (these change often — flag which ones look "temporary" vs. permanent allow/deny lists), IPAddressVersion, and every regex string. *Needed in CFN:* `AWS::WAFv2::IPSet` and `AWS::WAFv2::RegexPatternSet` — referenced by ARN inside Rule Group / Web ACL rule statements, so these must deploy **before** the Rule Groups that reference them.

### 2.7 Logging configuration

**Console:** WAF & Shield → Web ACL → Logging and metrics tab.

**CLI:**
```bash
aws wafv2 get-logging-configuration --resource-arn <web-acl-arn> --region <region>
```
*Document:* destination (Kinesis Firehose ARN / S3 / CloudWatch Logs), redacted fields, logging filter (allow/block/count-only conditions).
*Needed in CFN:* `AWS::WAFv2::LoggingConfiguration`. If the Firehose delivery stream was also created manually, discover it too (`aws firehose describe-delivery-stream`).

### 2.8 Resource associations

**CLI:**
```bash
aws wafv2 list-resources-for-web-acl --web-acl-arn <arn> --resource-type APPLICATION_LOAD_BALANCER --region <region>
aws fms list-compliance-status --policy-id <policy-id>
```
*Document:* which ALBs/CloudFront distributions/API Gateways are protected — cross-reference against the FMS policy scope to confirm nothing is protected "by accident" outside FMS control (manually-associated resources are a common finding and a migration risk — see Part 3).

### 2.9 Compliance

**Console:** Firewall Manager → Security policies → select policy → Compliance tab (shows compliant/non-compliant accounts and reasons).

**CLI:**
```bash
aws fms get-compliance-detail --policy-id <policy-id> --member-account <account-id>
```
*Document:* baseline compliance state **before** migration begins, per account. This is your rollback success criterion — post-migration compliance must match or exceed this baseline.

### 2.10 Organizations structure

**CLI:**
```bash
aws organizations list-roots
aws organizations list-organizational-units-for-parent --parent-id <root-id>
aws organizations list-accounts-for-parent --parent-id <ou-id>
aws organizations list-tags-for-resource --resource-id <account-id>
```
*Document:* full OU tree, which OU each of the 44 accounts sits in, and any account-level tags used in FMS `ResourceTag` exclusions.

### 2.11 Network Firewall (if applicable)

**CLI:**
```bash
aws network-firewall list-firewalls
aws network-firewall describe-firewall --firewall-name <name>
aws network-firewall list-rule-groups
aws network-firewall describe-rule-group --rule-group-name <name> --type STATEFUL
aws network-firewall describe-firewall-policy --firewall-policy-name <name>
```
*Document:* stateless default actions, stateful engine options (strict order vs. default), rule group references and priorities, subnet mappings, firewall endpoint IDs (needed for route table reconciliation post-migration).

---

## PART 3 — Migration Planning

| Phase | What happens | Why it's necessary |
|---|---|---|
| **Discovery** | Part 2 activities, exhaustively, for every region in scope | You cannot write an accurate CFN template for a resource you haven't fully inventoried. Missing even one custom rule silently changes production security posture. |
| **Documentation** | Convert raw CLI output into a structured design doc (per-resource property tables, an OU/account scope map, a dependency graph) | Creates a reviewable artifact for stakeholders and a rollback reference independent of the live console state. |
| **Backup** | Export every `get-*` CLI response to versioned JSON files in S3/git; snapshot compliance status | If a migration step goes wrong, you need the exact pre-migration configuration to manually restore, not just "what CloudFormation thinks it should be." |
| **Risk assessment** | Classify each resource: Low risk (import-eligible, e.g., IP Sets), Medium (Rule Groups — importable but capacity-locked), High (FMS Policy — cannot be imported cleanly, requires cutover strategy) | Determines which resources go through CFN's native `import` workflow vs. which need a parallel-build-then-cutover approach. |
| **Change management** | Formal CAB/change ticket, maintenance window, named approver, communication to app teams whose ALBs/CloudFront sit behind these WAFs | FMS policy changes affect all 44 accounts simultaneously; an untested change can 403 legitimate production traffic org-wide. |
| **Rollback planning** | For each phase: an explicit "how do I undo this in under 15 minutes" answer, including keeping the manually-created objects untouched until CFN is proven | Firewall/security tooling failure modes are binary and immediate (traffic blocked or unprotected) — rollback speed matters more than in most IaC migrations. |
| **Validation** | Dry-run in a non-prod account/OU first; diff CFN template `describe-stack-resources` output against Part 2 documentation field-by-field | Confirms the template reproduces the *exact* existing behavior before it touches production. |
| **Production deployment** | Staged rollout: import Rule Groups/IP Sets → validate → deploy new parallel Web ACL/FMS Policy → shadow/count mode → cutover → decommission manual resources | Minimizes blast radius; count-mode-first lets you validate rule matches against real traffic without blocking anything. |
| **Post-deployment verification** | Re-run Part 2 discovery commands against the now-CFN-managed resources; compare to backup; confirm FMS compliance = 100% across all 44 accounts | Confirms IaC state equals prior manual state (or an intentionally improved state) with no regression. |

---

## PART 4 — CloudFormation Design

### 4.1 Resource-by-resource

| Resource | CFN Type | Deploy order | Required properties | Common mistakes | Import support |
|---|---|---|---|---|---|
| IP Set | `AWS::WAFv2::IPSet` | 1 | `Name`, `Scope`, `IPAddressVersion`, `Addresses` | Forgetting `Scope: CLOUDFRONT` sets must deploy in `us-east-1` regardless of your primary region | ✅ Yes — `IMPORT` changeset, needs `Name`+`Id`+`Scope` |
| Regex Pattern Set | `AWS::WAFv2::RegexPatternSet` | 1 | `Name`, `Scope`, `RegularExpressionList` | Regex syntax differs slightly from PCRE — validate with `wafv2 check-capacity` equivalent testing before deploy | ✅ Yes |
| Rule Group | `AWS::WAFv2::RuleGroup` | 2 (after IP Sets/Regex Sets) | `Name`, `Scope`, `Capacity`, `Rules` | `Capacity` is immutable after creation — under-provisioning forces a full replace later; must be calculated with AWS's WCU formula, not guessed | ✅ Yes, but Capacity must match exactly or import fails |
| Web ACL (only for resources actually owned in Security Infra account, not FMS-managed ones) | `AWS::WAFv2::WebACL` | 3 | `Name`, `Scope`, `DefaultAction`, `VisibilityConfig`, `Rules` | Mixing managed-rule-group statements and rule-group-reference statements incorrectly in the `Rules` array — priorities must be globally unique across the whole array | ✅ Yes for standalone ACLs; **not applicable** to FMS-managed ACLs (see 4.2) |
| Logging Configuration | `AWS::WAFv2::LoggingConfiguration` | 4 | `ResourceArn`, `LogDestinationConfigs` | Firehose delivery stream name must start with `aws-waf-logs-` or logging silently fails | ✅ Yes |
| Firewall Manager Policy | `AWS::FMS::Policy` | 5 (last — references everything above) | `PolicyName`, `ResourceType` or `ResourceTypeList`, `SecurityServicePolicyData`, `ExcludeResourceTags`, `RemediationEnabled` | Forgetting `PolicyName` must be globally unique per account/service; `IncludeMap`/`ExcludeMap` OU IDs typo'd silently produce zero scoped accounts | ❌ **No native CFN import for `AWS::FMS::Policy`** — must be recreated (see 4.2 workaround) |
| Network Firewall Rule Group | `AWS::NetworkFirewall::RuleGroup` | 2 | `RuleGroupName`, `Type`, `Capacity`, `RuleGroup` (Suricata or structured) | Same Capacity-is-immutable trap as WAF | ✅ Yes |
| Network Firewall Policy | `AWS::NetworkFirewall::FirewallPolicy` | 3 | `FirewallPolicyName`, `FirewallPolicy` (stateless/stateful rule group refs + priorities) | Ordering of `StatefulRuleGroupReferences` matters if `STRICT_ORDER` engine option is set | ✅ Yes |

### 4.2 The `AWS::FMS::Policy` import gap — critical for your migration

Unlike most resources, **AWS::FMS::Policy cannot be imported into a CloudFormation stack** using the standard `CreateChangeSet --change-set-type IMPORT` workflow, because FMS treats the policy as an Organizations-service-managed object rather than a plain resource with importable metadata in all cases, and CFN's resource import feature has known gaps for `AWS::FMS::Policy`. Practical workaround (parallel-build-then-cutover):

1. Build the **new** FMS policy via CloudFormation with a **different `PolicyName`** (e.g., append `-v2`), scoped initially to a single non-prod OU only.
2. Validate compliance and Web ACL behavior in that OU.
3. Once validated, widen `IncludeMap` in the CFN template to cover the full 44-account scope, one OU/wave at a time (see Part 6 for staged rollout).
4. Once the CFN-managed policy shows 100% compliance across all accounts, delete the original manually-created FMS policy from the console (Firewall Manager auto-cleans up the old FMS-managed Web ACLs it created, if `DeleteAllPolicyResources=true` is set on deletion).

This means for a short window you will have **two** FMS policies active. Plan the deletion of the old one as an explicit, approved step in Part 6 — do not let it linger.

### 4.3 Recommended folder / stack structure

```
fms-iac/
├── templates/
│   ├── 01-ipsets-and-regex.yaml        # AWS::WAFv2::IPSet / RegexPatternSet
│   ├── 02-rulegroups-waf.yaml          # AWS::WAFv2::RuleGroup (customer rules)
│   ├── 03-rulegroups-networkfirewall.yaml
│   ├── 04-webacl-standalone.yaml       # only if any non-FMS-managed Web ACLs exist
│   ├── 05-logging.yaml                 # AWS::WAFv2::LoggingConfiguration
│   ├── 06-fms-policy-waf.yaml          # AWS::FMS::Policy (WAF type)
│   └── 07-fms-policy-networkfirewall.yaml
├── parameters/
│   ├── dev.json
│   ├── nonprod.json
│   └── prod.json
├── scripts/
│   ├── discover.sh                     # runs Part 2 CLI calls, dumps to json/
│   └── validate-diff.py                # diffs live config vs template render
├── docs/
│   ├── discovery-export/               # raw JSON backups per Part 2
│   ├── design-doc.md
│   └── rollback-runbook.md
└── pipeline/
    └── buildspec.yml / azure-pipelines.yml
```

Use nested stacks or a stack-per-template approach (not one monolith) so that Rule Group updates don't force a no-op redeploy risk on the FMS Policy stack, and so the higher-risk `AWS::FMS::Policy` stack can be deployed/rolled back independently.

---

## PART 5 — UI Walkthrough

### 5.1 WAF path, Security Infra account

```
Security Infra Account
 → Firewall Manager
   → Security policies
     → [select policy name]
       → Overview tab: ResourceType, scope summary
       → Policy detail tab: Web ACL rule template (this is what gets pushed)
       → Scope tab: included/excluded OUs, accounts, tags
       → Compliance tab: per-account compliant/non-compliant + reason
```

**Tracing into a member account:**
```
Compliance tab → click a specific member account row
 → shows the FMS-managed Web ACL name/ARN created in that account
 → "View in account" link (if console access to that account is available)
```
In the member account itself:
```
Member Account
 → WAF & Shield → Web ACLs → find FMManagedWebACLV2-<policy>-<id>
   → Rules tab: confirms it matches the central template (read-only — editing here is blocked/reverted by FMS)
   → Associated AWS resources tab: confirms which ALB/CloudFront/API Gateway is protected
   → Logging and metrics tab: confirms logging destination
 → EC2 → Load Balancers (if ALB) → select ALB → Integrated services tab: shows the WAF Web ACL currently attached — cross-check it matches
```

### 5.2 Network Firewall path (if applicable)

```
Security Infra Account
 → Firewall Manager → Security policies → [Network Firewall policy]
   → Scope tab: VPC/OU scope
   → Firewall Policy tab: stateless default actions, stateful rule group order
```
Member account:
```
Member Account
 → VPC → Network Firewall → Firewalls → [firewall name, FMS-created]
   → Details tab: firewall policy attached, subnet mappings
   → Firewall endpoints: one per AZ — note endpoint IDs
 → VPC → Route Tables → subnet route table → confirm FMS inserted the route to the firewall endpoint (0.0.0.0/0 → vpce-xxxx)
```

**Key difference between WAF and Network Firewall in the console:** WAF associations are visible directly on the protected resource (ALB/CloudFront) under "Integrated services" or "WAF" tab. Network Firewall associations are visible only via **route table** inspection — there's no equivalent "this VPC is protected by X" badge on the VPC itself, so route table review is a mandatory verification step for Network Firewall migrations that WAF migrations don't require.

---

## PART 6 — CloudFormation Deployment Sequence

```
1. Discovery (Part 2)              — export everything to docs/discovery-export/
2. Documentation (Part 3)          — design-doc.md complete, peer reviewed
3. Template creation (Part 4)      — one stack per resource layer
4. cfn-lint + cfn-guard validation — catch schema errors before any API call
5. Deploy to a throwaway/dev account — full stack, isolated OU, no member impact
6. Deploy Rule Groups/IP Sets to Security Infra via IMPORT changeset
   (these ARE importable — do this first since it's low-risk)
7. Deploy NEW FMS Policy (v2 name) scoped to ONE non-prod OU only
8. Validate: compliance = 100% in that OU, Web ACL rules match template exactly
9. Create change set widening scope to next OU wave; review diff; execute
10. Repeat wave-by-wave until all 44 accounts covered by the CFN-managed policy
11. Final validation: re-run Part 2 discovery against new policy, diff vs. baseline
12. Delete the original manually-created FMS policy (with DeleteAllPolicyResources)
13. Post-deployment verification (Part 3) + close change ticket
```

**Why order matters:** IP Sets and Regex Pattern Sets must exist before Rule Groups reference them (CFN dependency); Rule Groups must exist before the FMS Policy's `SecurityServicePolicyData` JSON can reference their ARNs; and the FMS Policy must be validated in a narrow scope before widening, because a scope-wide policy error blocks/allows traffic for **all 44 accounts at once** — there is no per-account canary unless you do the OU-wave approach manually.

**Rollback per step:**
- Steps 6 (Rule Groups/IP Sets import): `cfn delete-stack` reverts to nothing-changed since these aren't referenced yet by anything live.
- Steps 7–10 (FMS Policy waves): delete the CFN stack (or shrink `IncludeMap` via changeset) — the old manually-created policy is still active and untouched, so member accounts fall back to it automatically with no traffic gap.
- Step 12 (deleting old policy): this is the **point of no easy return** — only execute after step 11 passes fully, and keep the Part 2 backup JSON as the manual-recreation fallback if something is discovered wrong after this point.

---

## PART 7 — Best Practices

- **Version control:** every template in Git (GitHub Enterprise, matching your existing CFN workflow); PRs required for changes to `06-fms-policy-waf.yaml` specifically, given blast radius.
- **CI/CD:** `cfn-lint` + `cfn-guard` (policy-as-code checks, e.g., "no FMS policy may set `RemediationEnabled: false` in prod") on every PR; deploy via pipeline, not manual `aws cloudformation deploy`.
- **StackSets:** not needed for the FMS Policy resource itself (Organizations-native fan-out handles that) — but useful if you also need to deploy account-level guardrails (e.g., an SCP preventing member accounts from manually detaching FMS-managed Web ACLs).
- **Drift detection:** run `aws cloudformation detect-stack-drift` on the Rule Group/IP Set stacks weekly — these are the resources most likely to be hand-edited by an on-call engineer during an incident.
- **Change sets:** mandatory before every production apply — never use `--no-fail-on-empty-changeset` blindly, and always eyeball the `IncludeMap`/`ExcludeMap` diff specifically.
- **CloudTrail:** enable org-wide trail capturing `fms.amazonaws.com` and `wafv2.amazonaws.com` events; this is your audit source for "who changed the policy" going forward, same pattern you used to trace the Route 53 drift.
- **AWS Config:** add a Config rule (or Conformance Pack) checking that ALBs/CloudFront distributions in scope have an associated Web ACL — a compliance backstop independent of FMS's own compliance view.
- **Monitoring/Logging:** CloudWatch alarms on WAF `BlockedRequests`/`CountedRequests` metrics; centralize WAF logs from all 44 accounts into one log-archive account via Firehose cross-account delivery.
- **Security / least privilege IAM:** the CFN deploy role needs `fms:PutPolicy`, `fms:DeletePolicy`, `wafv2:*RuleGroup`, `wafv2:*IPSet`, `wafv2:*WebACL` scoped to specific ARNs where wildcarding isn't required — do not grant `fms:*` broadly.
- **Cross-account deployment:** the FMS Policy stack must deploy from the Security Infra account specifically; use a dedicated CI/CD role assumed into that account only, not your general multi-account deploy role.
- **Multi-region:** `CLOUDFRONT` scope resources must deploy to `us-east-1`; `REGIONAL` resources need one stack instance per region you protect ALBs/API Gateways in.
- **Disaster recovery:** keep the Part 2 discovery-export JSON as a versioned, immutable backup (S3 with versioning + Object Lock) independent of the CFN state file, so you can manually reconstruct the pre-migration state even if CloudFormation and Git are both unavailable.

---

## PART 8 — ASCII Diagrams

**Overall architecture**
```
                    ┌────────────────────────┐
                    │  Management Account      │
                    │  (enables FMS org-wide)  │
                    └────────────┬─────────────┘
                                 │ delegates
                    ┌────────────▼─────────────┐
                    │  Security Infra Account   │
                    │  (delegated admin)        │
                    │  Rule Groups / IP Sets /  │
                    │  FMS Policy (CFN-managed) │
                    └────────────┬─────────────┘
             ┌───────────────────┼───────────────────┐
     ┌───────▼──────┐   ┌────────▼───────┐   ┌───────▼──────┐
     │ Member Acct   │   │ Member Acct    │   │ Member Acct  │
     │ (OU: Prod)    │   │ (OU: NonProd)  │   │ (OU: Sandbox)│
     │ FMS Web ACL   │   │ FMS Web ACL    │   │ FMS Web ACL  │
     └───────────────┘   └────────────────┘   └──────────────┘
```

**FMS deployment flow**
```
CFN template applied in Security Infra
        │
        ▼
AWS::FMS::Policy created/updated
        │
        ▼
FMS evaluates Organizations OU/account scope (IncludeMap/ExcludeMap)
        │
        ▼
For each in-scope account+region:
        │  create/update FMManagedWebACLV2
        │  associate to matching ResourceType resources
        │  write compliance record
        ▼
Compliance dashboard updates (Security Infra account, read-only rollup)
```

**Rule → Rule Group → Web ACL → FMS Policy → Member Account → Protected Resource**
```
Rule ──► RuleGroup ──► WebACL template (inside FMS Policy) ──► FMS::Policy
                                                                    │
                                                          pushed org-wide
                                                                    │
                                            ┌───────────────────────┴───┐
                                            ▼                            ▼
                                   Member Account Web ACL      Member Account Web ACL
                                            │                            │
                                            ▼                            ▼
                                    Associated ALB/CF                Associated ALB/CF
```

**CloudFormation deployment dependency graph**
```
IPSet ──┐
        ├──► RuleGroup ──► FMS::Policy (SecurityServicePolicyData references RuleGroup ARN)
RegexPatternSet ──┘              │
                                  ▼
                        LoggingConfiguration (attached post-creation, references FMS-managed ACL ARN via custom lookup / SSM if needed)
```

**Request processing flow (WAF)**
```
Client → CloudFront/ALB/APIGW → FMS-managed Web ACL
              │
              ├─ Rule Group 1 (managed) evaluated
              ├─ Rule Group 2 (customer) evaluated
              ├─ Rate-based rule evaluated
              │
              ▼
        Default action
        ALLOW → origin        BLOCK → 403 + WAF log entry (Firehose → S3)
```
