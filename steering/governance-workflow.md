# Governance Control Workflow

## Purpose
This steering file guides the agent through the end-to-end governance control workflow — from understanding operator intent to presenting a deployment plan.

## Phase 0: Environment Overview (Automatic)

**This phase runs IMMEDIATELY after onboarding/credential verification, BEFORE asking for operator intent.**

The operator needs to see their environment to make informed decisions. After confirming credentials are valid:

1. Discover the org structure, CT home region, governed regions
2. Check landing zone version and drift status
3. Check Security Hub delegated admin and CSPM
4. List all OUs with registration status
5. List all accounts with enrollment status
6. **Display the full environment summary** (see `environment-discovery.md` template)

**Then** ask for their governance intent with the OU table visible.

This means the operator sees something like:

```
═══════════════════════════════════════════════════════════
ENVIRONMENT SUMMARY
═══════════════════════════════════════════════════════════

Organization: o-bmlaerlczb
Management Account: 386986622918
Control Tower Home Region: ap-southeast-2
Governed Regions: ap-southeast-4, ap-southeast-2, us-east-1, us-west-2
Landing Zone Version: 4.0 [✓ latest]
Landing Zone Status: ACTIVE [✓ in sync, no drift]

Security Hub:
- Delegated Admin: 111111111111 (Security-Audit)
- CT CSPM Standard: ✓ Enabled

Organizational Units:
┌────────────────────────────────────────────────────────────────────┐
│ # │ OU Name      │ OU ID              │ Accounts │ Controls │ CT Status    │
├───┼──────────────┼────────────────────┼──────────┼──────────┼──────────────┤
│ 1 │ OUConfigOnly │ ou-l3h0-1uwdx2wr   │ 1        │ 9        │ ✓ Registered │
│ 2 │ OU1          │ ou-l3h0-42achbcy   │ 2        │ 14       │ ✓ Registered │
│ 3 │ Services     │ ou-l3h0-6uq759vq   │ 1        │ 0        │ ✓ Registered │
│ 4 │ Sandbox      │ ou-l3h0-ap6vlll9   │ 1        │ 0        │ ✓ Registered │
│ 5 │ Security     │ ou-l3h0-yra8dvgc   │ 2        │ 12       │ ✓ Registered │
└────────────────────────────────────────────────────────────────────┘

Accounts: 7 total (all ACTIVE, all enrolled)
═══════════════════════════════════════════════════════════

What governance control would you like to implement?
Please specify:
1. What should be controlled (e.g., EBS encryption, public S3, required tags)
2. Which OU (by number or name above)
3. Detective (detect violations) / Preventive (block creation) / Both
```

**Key principle**: Show the environment FIRST, then ask the question. Don't ask "which OU?" and then go discover what OUs exist.

---

## Phase 1: Understand Operator Intent

**⚠️ CRITICAL: Gather ALL required inputs from the operator BEFORE doing any API calls or research.**

When the operator describes a governance need, you MUST collect these inputs before proceeding:

### Required Inputs (ask upfront)

1. **What resource type** is being governed (S3, EC2, RDS, IAM, etc.)
2. **What property** must be controlled (encryption, public access, tagging, network, etc.)
3. **What enforcement level** is needed:
   - **Preventive** — block non-compliant resources from being created
   - **Detective** — detect existing non-compliant resources
   - **Proactive** — validate CloudFormation templates before deployment
   - **All of the above** (defense-in-depth, recommended)
4. **What scope (target OU)** — which OU(s) should this control apply to?
5. **What exceptions** — break-glass roles, specific accounts, specific use cases

### ⚠️ DO NOT proceed to Phase 2 until you have answers to items 1-4.

If the operator provides a clear statement like "detect unencrypted EBS volumes" but doesn't specify the target OU, **immediately ask which OU(s) to target** before doing any environment research:

```
Before I research the environment, I need to know:

1. Which OU(s) should this control be applied to?
   - All OUs (organization-wide)
   - Specific OUs (please name them)
   - "I'm not sure" → I'll discover your org structure and show you the OUs to choose from

2. Should this be a hard block (preventive) or just detection/reporting (detective)?
```

**If the operator says "I'm not sure" about the OU**, or if onboarding has already completed and you have the org structure:
- Show the environment summary WITH the OU/account table (see `environment-discovery.md` Environment Summary Template)
- Let the operator pick from the table
- Then proceed to Phase 2 scoped to their selection

### Why Ask First

The target OU determines:
- Which existing controls to check for duplicates (saves unnecessary API calls)
- Whether to recommend a Control Tower catalog control (which applies per-OU)
- The blast radius of the deployment
- Whether a sandbox-first rollout is appropriate

### Clarifying Questions Template

Ask these questions in ONE message, not spread across the conversation:

```
To implement your governance control, I need a few details:

1. Target OU(s): Which organizational unit(s) should this apply to?
   (e.g., all production OUs, sandbox first, specific OU name)

2. Enforcement type: Should this...
   a. Detect existing non-compliant resources (detective)
   b. Block creation of non-compliant resources (preventive)
   c. Both (recommended for defense-in-depth)

3. Exceptions: Are there any roles or accounts that should be exempt?

Once I have these, I'll check your environment for existing controls and recommend the best approach.
```

### Intent Examples

| Operator Says | Resource | Property | Enforcement | **Ask** |
|---|---|---|---|---|
| "Ensure S3 buckets are encrypted" | S3 | Encryption | ? | Which OU? Detective or preventive? |
| "Block public RDS in production" | RDS | Public access | Preventive | Which OU is "production"? |
| "Require tags on EC2 everywhere" | EC2 | Tags | ? | All OUs? Which tags? Detective or proactive? |
| "Detect unencrypted EBS volumes on Sandbox" | EC2/EBS | Encryption | Detective | ✅ Complete — proceed |

## Phase 2: Environment Discovery

**⚠️ PRE-CONDITION: Only begin this phase after the operator has confirmed the target OU(s) and enforcement type.**

If you don't have the target OU yet, go back to Phase 1 and ask.

Once intent is clear, discover the current state. Load `steering/environment-discovery.md` for the detailed discovery procedure.

**⚠️ FIRST ACTION: Determine the Control Tower home region.** All subsequent Control Tower and StackSet API calls depend on this. See `environment-discovery.md` Step 2.

**Summary of what to discover:**
1. Organization structure (root, OUs, accounts)
2. **Control Tower home region and governed regions** ← MUST BE FIRST
3. Currently enabled Control Tower controls **on the target OU only** (using home region!)
4. Existing SCPs attached to **the target OU only**
5. Existing Config rules (using `aws___call_aws` per governed region — NOT run_script)
6. Check if the requested control is ALREADY implemented **on the target OU**

**Efficiency note**: Since you already know the target OU, only query that OU for existing controls — don't enumerate controls on every OU in the organization.

**If the control already exists**, inform the operator:
- "A control matching your intent already exists: [control name/identifier]. It is currently [enabled/disabled] on [scope]. No action needed."
- Offer to show the control details or extend its scope if needed.

## Phase 3: Control Selection

Load `steering/environment-discovery.md` results, then determine the best control approach.

### Step 3.0: Check Organizations Policy Quotas (BEFORE recommending SCP/RCP)

**Check quotas BEFORE proposing any SCP or RCP.** If the target OU is at or near the limit, an SCP/RCP approach won't work.

**Quota limits (hard limits, per entity):**

| Policy Type | Max per Root | Max per OU | Max per Account |
|---|---|---|---|
| SCP | 10 | 10 | 10 |
| RCP | 5 | 5 | 5 |

**⚠️ Control Tower consideration:** AWS Control Tower's own preventive controls consume SCP slots. In practice, an OU managed by Control Tower may already have several CT-managed SCPs attached. Check the actual count before adding more.

**Check current SCP count on target OU:**
```
aws organizations list-policies-for-target --target-id <OU_ID> --filter SERVICE_CONTROL_POLICY
```

**Check current RCP count on target OU:**
```
aws organizations list-policies-for-target --target-id <OU_ID> --filter RESOURCE_CONTROL_POLICY
```

**If at or near limit, warn the operator:**

```
⚠️  POLICY QUOTA WARNING

Target OU: [OU name] ([OU ID])

┌────────────────────────────────────────────────────────┐
│ Policy Type │ Attached │ Limit │ Available │ Status    │
├─────────────┼──────────┼───────┼───────────┼───────────┤
│ SCP         │ 8        │ 10    │ 2         │ ⚠️ Low    │
│ RCP         │ 4        │ 5     │ 1         │ ⚠️ Low    │
└────────────────────────────────────────────────────────┘

[If at limit]:
❌ Cannot attach another [SCP/RCP] to this OU — quota reached (10/10).

Options:
1. Consolidate existing SCPs (merge multiple into one)
2. Use a nested OU structure (child OUs get additional SCP slots)
3. Use a different control type (proactive CloudFormation Guard instead of preventive SCP)
4. Target a different OU

[If near limit (≤2 remaining)]:
⚠️  Only [N] SCP slot(s) remaining on this OU.
    Proceeding will leave [N-1] for future controls.
    Consider consolidating existing SCPs first.
```

**Decision logic:**
- If SCP quota is full → cannot use SCP; suggest proactive (CFN Guard/Hook) or consolidation
- If RCP quota is full → cannot use RCP; suggest SCP alternative
- If quota is low (≤2 remaining) → warn but allow, suggest consolidation as a follow-up

### Step 3.1: Search Control Tower Catalog

Use `aws___search_documentation` to find matching Control Tower controls:
```
Search for: "Control Tower control [resource type] [property]"
Topics: ["reference_documentation"]
```

Use `aws___call_aws` to list available controls:
```
aws controltower list-controls
```

Filter controls by:
- `ControlScope` matching the target resource
- `Behavior` matching the enforcement level (PREVENTIVE, DETECTIVE, PROACTIVE)

### Step 3.2: Evaluate Catalog Controls

For each candidate control:
1. Read its description and check if it fully covers the operator's intent
2. Check if it's already enabled on the target OUs
3. Assess if the control's scope matches (too broad? too narrow?)

### Step 3.3: Present Recommendation

**If catalog control exists:**
```
I found a Control Tower catalog control that matches your intent:

- Control: [control identifier]
- Name: [human-readable name]
- Type: [Preventive/Detective/Proactive]
- Description: [what it does]
- Behavior: [how it enforces]

This is an AWS-managed control that integrates with Control Tower drift detection.

Would you like me to enable this control on [target OU(s)]?
```

**If no catalog control exists:**
```
No Control Tower catalog control fully covers your requirement.
Checking AWS Config managed rules next...
```

## Phase 3b: Check AWS Config Managed Rules

**Before going to custom authoring, check if an AWS Config managed rule meets the need.** AWS provides 300+ managed rules that require no custom code — they just need to be deployed via StackSet.

### Why Check Managed Rules

| Approach | Effort | Maintenance | Drift Detection |
|---|---|---|---|
| CT catalog control | Zero (AWS-managed) | AWS maintains | ✓ Built-in |
| **Config managed rule** | **Low (deploy via StackSet)** | **AWS maintains rule logic** | ✓ Config evaluates |
| Custom rule (Guard/Lambda) | High (write + test) | You maintain | Partial |

### Step 3b.1: Search for a Matching Managed Rule

Use `aws___search_documentation` to find managed Config rules:
```
Search: "AWS Config managed rule [resource] [property]"
Topics: ["reference_documentation"]
```

Or search the Config rule reference directly:
```
Search: "Config rule identifier [ENCRYPTED_VOLUMES | RDS_INSTANCE_PUBLIC_ACCESS_CHECK | etc]"
```

### Common Managed Rules Reference

| Intent | Managed Rule Identifier | Resource Type |
|---|---|---|
| EBS encryption | `ENCRYPTED_VOLUMES` | AWS::EC2::Volume |
| EBS encryption by default | `EC2_EBS_ENCRYPTION_BY_DEFAULT` | Account-level |
| S3 bucket encryption | `S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED` | AWS::S3::Bucket |
| S3 public read | `S3_BUCKET_PUBLIC_READ_PROHIBITED` | AWS::S3::Bucket |
| S3 public write | `S3_BUCKET_PUBLIC_WRITE_PROHIBITED` | AWS::S3::Bucket |
| RDS public access | `RDS_INSTANCE_PUBLIC_ACCESS_CHECK` | AWS::RDS::DBInstance |
| RDS encryption | `RDS_STORAGE_ENCRYPTED` | AWS::RDS::DBInstance |
| EC2 no public IP | `EC2_INSTANCE_NO_PUBLIC_IP` | AWS::EC2::Instance |
| Required tags | `REQUIRED_TAGS` | Multiple (configurable) |
| Unrestricted SSH | `INCOMING_SSH_DISABLED` | AWS::EC2::SecurityGroup |
| Unrestricted RDP | `RESTRICTED_INCOMING_TRAFFIC` | AWS::EC2::SecurityGroup |
| IAM root MFA | `ROOT_ACCOUNT_MFA_ENABLED` | Account-level |
| IAM user no policies | `IAM_USER_NO_POLICIES_CHECK` | AWS::IAM::User |
| CloudTrail enabled | `CLOUD_TRAIL_ENABLED` | Account-level |
| VPC flow logs | `VPC_FLOW_LOGS_ENABLED` | AWS::EC2::VPC |

### Step 3b.2: Evaluate Managed Rule

If a managed rule exists:
1. Verify it checks the exact property the operator wants
2. Check if it accepts parameters (e.g., `REQUIRED_TAGS` takes tag key names)
3. Confirm the resource type scope matches

### Step 3b.3: Present Recommendation

**If a managed rule matches:**
```
I found an AWS Config managed rule that matches your intent:

- Rule Identifier: [ENCRYPTED_VOLUMES]
- Description: [what it checks]
- Resource Type: [AWS::EC2::Volume]
- Parameters: [none / list params]
- Maintenance: AWS-managed (no custom code needed)

This is NOT a Control Tower catalog control, so I'll deploy it via
CloudFormation StackSet to all governed regions.

Would you like me to prepare the deployment plan?
```

**If no managed rule matches:**
```
No AWS Config managed rule covers your requirement.
Proceeding to custom control authoring (Phase 4)...
```

### CloudFormation Template for Managed Rule Deployment

**Deployment options for Config managed rules:**

| Method | Deployed From | Scope | OU Targeting |
|---|---|---|---|
| **Organization Config Rule** | Management account or Config delegated admin | **Entire organization** (all accounts) | ❌ No OU targeting — can only exclude individual accounts |
| **StackSet** | Management account (CT home region) | Specific OUs/accounts | ✅ Precise OU targeting |

**Option A: Organization Config Rule (org-wide only)**

Applies to **ALL accounts in the organization**. You CANNOT target a specific OU. You can only exclude individual accounts.

Use this when:
- The operator wants the rule applied organization-wide
- The operator says "all OUs" or "everywhere"

Do NOT use this when:
- The operator specified a single OU (e.g., "only on Sandbox")
- The operator wants to test in one OU before rolling out

Deploy from the **management account** (or Config delegated admin if configured):

```
aws configservice put-organization-config-rule \
  --organization-config-rule-name "kiro-power-<rule-name>" \
  --organization-managed-rule-metadata \
    RuleIdentifier=ENCRYPTED_VOLUMES,\
    Description="Deployed by kiro-power-governance-controls" \
  --excluded-accounts '["<management-account-id>"]' \
  --region <HOME_REGION>
```

**Check if Config has a delegated admin:**
```
aws organizations list-delegated-administrators --service-principal config-multiaccountsetup.amazonaws.com
```

If a delegated admin exists, the organization Config rule MUST be created from that account, not the management account.

**Option B: StackSet deployment (PREFERRED for specific OU targeting)**

Use this when the operator specified a particular OU. Deploys from the **management account** in the CT home region. Stack instances run in each member account within the targeted OU.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: !Sub "kiro-power - Config managed rule: ${RuleIdentifier}"

Parameters:
  RuleIdentifier:
    Type: String
  RuleName:
    Type: String

Resources:
  ConfigRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: !Ref RuleName
      Description: !Sub "Deployed by kiro-power-governance-controls"
      Source:
        Owner: AWS
        SourceIdentifier: !Ref RuleIdentifier
      Scope:
        ComplianceResourceTypes:
          - "AWS::EC2::Volume"  # Adjust per rule
```

**Decision guide:**
- Operator says "all OUs" / "organization-wide" → **Organization Config Rule**
- Operator specifies a single OU or subset → **StackSet** (precise targeting)
- Config delegated admin exists → Organization rules must come from that account

---

## Phase 4: Custom Control Authoring

Load `steering/custom-control-authoring.md` for detailed authoring guidance.

## Phase 5: Deployment Planning

Load `steering/stackset-deployment.md` for deployment procedures.

### Deployment Summary Template

Before ANY deployment, present this summary to the operator:

```
═══════════════════════════════════════════════════════════
GOVERNANCE CONTROL DEPLOYMENT PLAN
═══════════════════════════════════════════════════════════

Intent: [Original operator request]

Controls to Deploy:
┌─────────────────────────────────────────────────────────┐
│ 1. [Control type]: [Name/Description]                   │
│    Scope: [OUs/Accounts]                                │
│    Regions: [list of governed regions]                   │
│    Method: [CT catalog enable / StackSet deploy]        │
├─────────────────────────────────────────────────────────┤
│ 2. [Control type]: [Name/Description]                   │
│    Scope: [OUs/Accounts]                                │
│    Regions: [list of governed regions]                   │
│    Method: [CT catalog enable / StackSet deploy]        │
└─────────────────────────────────────────────────────────┘

Deployment Details:
- Source Region: [Control Tower home region]
- Target Regions: [all governed regions]
- StackSet Name: [name]
- Permission Model: SELF_MANAGED
- Execution Role: AWSControlTowerExecution
- Max Concurrent: 10%
- Failure Tolerance: 0%

⚠️  Impact:
- [X] accounts across [Y] OUs will be affected
- New resources that violate this control will be [blocked/flagged]
- Existing non-compliant resources will be [detected and reported]

═══════════════════════════════════════════════════════════
Proceed with deployment? (yes/no)
═══════════════════════════════════════════════════════════
```

**NEVER deploy without explicit "yes" from the operator.**

## Phase 6: Post-Deployment — Defense-in-Depth Recommendation

**After successfully enabling a control, ALWAYS recommend complementary controls for defense-in-depth.**

A single control type leaves gaps:
- **Detective only** → finds violations but doesn't prevent new ones from being created
- **Preventive only** → blocks new violations but doesn't find existing non-compliant resources
- **Proactive only** → catches CloudFormation deployments but misses console/CLI/SDK-created resources

### Recommendation Template

After confirming a detective control is live, present:

```
═══════════════════════════════════════════════════════════
✅ DETECTIVE CONTROL ENABLED SUCCESSFULLY

Control: [name]
OU: [target]
Regions: [all governed regions]

This will detect existing non-compliant resources and report them.
However, it will NOT prevent new non-compliant resources from being created.

═══════════════════════════════════════════════════════════
🛡️  DEFENSE-IN-DEPTH RECOMMENDATION

For full coverage, consider adding a complementary control:

┌─────────────────────────────────────────────────────────┐
│ Option │ Type        │ What it does                     │
├────────┼─────────────┼──────────────────────────────────┤
│ A      │ Preventive  │ Block creation of non-compliant  │
│        │ (SCP)       │ resources via API                │
├────────┼─────────────┼──────────────────────────────────┤
│ B      │ Proactive   │ Reject non-compliant             │
│        │ (CFN Hook)  │ CloudFormation deployments       │
├────────┼─────────────┼──────────────────────────────────┤
│ C      │ Both A + B  │ Full defense-in-depth            │
└─────────────────────────────────────────────────────────┘

[If a catalog preventive/proactive control exists]:
→ CT catalog control available: [control name] ([identifier])
  Type: [Preventive/Proactive]
  Would you like me to enable it on the same OU?

[If no catalog control exists]:
→ No catalog control available for prevention.
  I can create a custom SCP to block [non-compliant action].
  Would you like me to draft it?

Or type "no thanks" if detective-only is sufficient for now.
═══════════════════════════════════════════════════════════
```

### After enabling a preventive control, recommend detective:

```
✅ PREVENTIVE CONTROL ENABLED

This blocks new non-compliant resources. However, existing resources
created before this control was enabled are NOT affected.

Would you like me to also enable a detective control to find and
report any existing non-compliant resources?
```

### After enabling a proactive control, recommend detective:

```
✅ PROACTIVE CONTROL ENABLED

This validates CloudFormation templates before deployment. However:
- Resources created via console/CLI/SDK bypass this check
- Existing non-compliant resources are not detected

Would you like me to also enable:
- A detective control (find existing violations)?
- A preventive SCP (block non-compliant API calls)?
```

### Control Pairing Reference

| If you just enabled... | Recommend adding... |
|---|---|
| Detective (Config rule) | Preventive (SCP) or Proactive (CFN Hook) |
| Preventive (SCP) | Detective (Config rule) |
| Proactive (CFN Hook) | Detective (Config rule) + Preventive (SCP) |

**Always search the Control Tower catalog first** for the complementary control before proposing custom authoring.
