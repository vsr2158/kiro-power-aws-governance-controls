---
name: "aws-governance-controls"
displayName: "AWS Governance Control Automation"
description: "Automates deployment of AWS governance controls - reviews multi-account environments, detects existing controls, recommends Control Tower catalog controls or creates custom preventive/detective/proactive controls, and deploys via CloudFormation StackSets"
keywords: ["governance", "control tower", "controls", "scp", "rcp", "config rule", "cloudformation guard", "stacksets", "compliance", "preventive", "detective", "proactive", "multi-account", "organizations", "landing zone"]
author: "AWS"
homepage: "https://github.com/awslabs/mcp"
repository: "https://github.com/vsr2158/kiro-power-aws-governance-controls"
---

# AWS Governance Control Automation

## Overview

This power enables cloud operators to describe their governance intent in natural language. The agent automates the full lifecycle from discovery to deployment.

### Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Operator: "I need to detect unencrypted EBS volumes on Prod OU"   │
│                                                                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
                 ┌──────────────────────────┐
                 │  Phase 1: Gather Intent   │
                 │  • What control?          │
                 │  • Which OU?              │
                 │  • Detective/Preventive?  │
                 └─────────────┬────────────┘
                               │
                               ▼
                 ┌──────────────────────────┐
                 │  Phase 2: Discover Env    │
                 │  • CT home region         │
                 │  • OU registration        │
                 │  • Account enrollment     │
                 │  • Security Hub/CSPM      │
                 │  • Existing controls      │
                 └─────────────┬────────────┘
                               │
                               ▼
                 ┌──────────────────────────┐
                 │  Phase 3: Check Catalog   │
                 │  • Search CT controls     │
                 │  • Check for duplicates   │
                 └─────────┬───────┬────────┘
                           │       │
              ┌────────────┘       └────────────┐
              ▼                                  ▼
┌──────────────────────┐          ┌──────────────────────────┐
│ Catalog match found  │          │  No catalog match        │
│ → Recommend to user  │          │                          │
│ → Enable on target   │          └────────────┬─────────────┘
│   OU after approval  │                       │
└──────────┬───────────┘                       ▼
           │                    ┌──────────────────────────┐
           │                    │  Phase 3b: Check Config   │
           │                    │  Managed Rules            │
           │                    │  • 300+ AWS managed rules │
           │                    │  • No custom code needed  │
           │                    │  • Deploy via StackSet    │
           │                    └─────────┬───────┬────────┘
           │                              │       │
           │                 ┌────────────┘       └────────────┐
           │                 ▼                                  ▼
           │   ┌──────────────────────┐          ┌──────────────────────┐
           │   │ Managed rule found   │          │  No managed rule     │
           │   │ → Deploy via         │          │  → Phase 4: Author   │
           │   │   StackSet           │          │    custom SCP/RCP/   │
           │   │                      │          │    Guard + Config    │
           │   └──────────┬───────────┘          └────────────┬────────┘
           │              │                                    │
           └──────────────┼────────────────────────────────────┘
                          │
                          ▼
                ┌──────────────────────────┐
                │  Phase 5: Deploy          │
                │  • Present plan           │
                │  • Wait for approval      │
                │  • Enable/deploy          │
                │  • Verify success         │
                └─────────────┬────────────┘
                              │
                              ▼
                ┌──────────────────────────┐
                │  Phase 6: Defense-in-     │
                │  Depth Recommendation     │
                │  • Suggest complementary  │
                │    control type           │
                │  • Detective→Preventive   │
                │  • Preventive→Detective   │
                └──────────────────────────┘
```

## Onboarding

### Prerequisites

**AWS credentials are required for ALL features of this power.**

This power operates against a multi-account AWS Organizations environment. You need:

1. **Management account credentials** — for Organizations, Control Tower, and StackSets operations
2. **Control Tower execution role** — `AWSControlTowerExecution` role must exist in member accounts (standard with Control Tower)

### Step 1: Verify AWS CLI and credentials

```bash
aws --version
# Must be AWS CLI 2.32.0 or later

aws sts get-caller-identity
# Must return the MANAGEMENT account identity
```

Verify you are in the management account:
```bash
aws organizations describe-organization --query 'Organization.MasterAccountId'
# This must match your current account ID
```

**Set your default AWS region:** The power's MCP server does NOT hardcode a region. It uses your AWS CLI's default region (from `~/.aws/config` or `AWS_DEFAULT_REGION`). 

The agent will **discover your Control Tower home region automatically** during Phase 2 and pass `--region` explicitly on all subsequent calls. You don't need to configure anything region-specific in the power.

### Step 2: Verify Control Tower is active

**Control Tower APIs are REGIONAL — they only work in the CT home region.**

Find your landing zone (try your expected home region):
```bash
aws controltower list-landing-zones --region ap-southeast-2
# If this fails with ResourceNotFoundException, try other regions:
# aws controltower list-landing-zones --region us-east-1
# aws controltower list-landing-zones --region eu-west-1
```

Get details and note the home region:
```bash
aws controltower get-landing-zone \
  --landing-zone-identifier <LANDING_ZONE_ARN> \
  --region <HOME_REGION> \
  --query 'landingZone.manifest.governedRegions'
```

**⚠️ ALL Control Tower and StackSet operations in this power will use the home region. The power will fail if it tries to call Control Tower APIs in the wrong region.**

### Step 3: Verify StackSets permissions

```bash
aws cloudformation list-stack-sets --status ACTIVE --query 'Summaries[0].StackSetName'
# Should succeed without permission errors
```

### Step 4: Verify the MCP server is connected

Confirm the `aws-mcp` MCP server is running and tools like `aws___call_aws` and `aws___run_script` are available. Run `/tools` or `/mcp` to verify.

If the MCP server is not connected, ensure your `mcp.json` is configured per the MCP server section below.

### Step 5: Auto-approve AWS tool calls

This power makes frequent AWS API calls during environment discovery and control deployment. To avoid repeated permission prompts, trust the AWS MCP tools:

**In Kiro IDE:** When prompted for tool approval, select **"Always allow"** or **"Trust for this session"** for:
- `aws___call_aws`
- `aws___run_script`
- `aws___get_presigned_url`

**In Kiro CLI:** Start with `--trust-tools "@aws-mcp"` or approve on first use and select trust.

Alternatively, add these to your workspace or global MCP settings (`~/.kiro/settings/mcp.json`) under the server's `autoApprove` list.

---

## When to Load Steering Files

- Initial environment overview (Phase 0) → `steering/environment-discovery.md`
- Understanding operator intent and planning control strategy → `steering/governance-workflow.md`
- Reviewing existing controls and detecting duplicates → `steering/environment-discovery.md`
- Creating custom preventive/detective/proactive controls → `steering/custom-control-authoring.md`
- Deploying controls via StackSets → `steering/stackset-deployment.md`

---

## Workflow Summary

### Phase 0: Environment Overview (runs automatically after onboarding)
After verifying credentials, **immediately** discover and display the environment summary including:
- Landing zone version, status, drift
- Control Tower home region and governed regions  
- Security Hub delegated admin and CSPM status
- All OUs with registration status, account count, and control count
- All accounts with enrollment status

**Display this BEFORE asking the operator for their intent.** This gives them context to make informed choices about target OU and scope.

### Phase 1: Gather Intent
The operator describes what governance control they need in natural language.

**Before proceeding to any API calls or research, the agent MUST ask:**
1. Which OU(s) should this apply to?
2. Should it be detective, preventive, or both?
3. Any exemptions needed?

**Do NOT start environment discovery until the target OU is confirmed.**

### Phase 2: Environment Discovery
- Query AWS Organizations structure (OUs, accounts)
- Identify Control Tower home region and governed regions
- List enabled controls in Control Tower
- List existing Config rules and SCPs
- Determine if the requested control already exists

### Phase 3: Control Selection
- Search the Control Tower controls catalog for matching controls
- Classify matches as Detective, Preventive, or Proactive
- If a catalog control exists, present it to the operator for approval
- If no catalog control provides the coverage, proceed to Phase 3b

### Phase 3b: Check AWS Config Managed Rules
- If no CT catalog control matches, check the 300+ AWS Config managed rules
- Managed rules require no custom code — just deploy via StackSet
- If a managed rule matches, prepare a StackSet deployment
- Only if no managed rule covers the need, proceed to custom authoring

### Phase 4: Custom Control Authoring (if needed)
- **Detective**: Create an AWS Config custom rule (Lambda-based or Guard-based)
- **Preventive**: Create an SCP or RCP that denies non-compliant resource creation
- **Proactive**: Create a CloudFormation Guard rule for pre-deployment validation
- Always create both a detective AND a preventive/proactive control for defense-in-depth

### Phase 5: Deployment
- Create CloudFormation templates for all controls
- Package into a StackSet deployed from the Control Tower home region
- Target all Control Tower governed regions
- Use `AWSControlTowerExecution` role for stack instances in member accounts
- Present a complete deployment summary to the operator BEFORE executing
- Deploy only after explicit operator approval

### Phase 6: Defense-in-Depth Recommendation
After successful deployment of any single control type, **always recommend complementary controls**:
- After enabling **detective** → propose preventive (SCP) or proactive (CFN Hook)
- After enabling **preventive** → propose detective (Config rule) to find existing violations
- After enabling **proactive** → propose detective + preventive for full coverage
- Search the Control Tower catalog first; propose custom only if no catalog match exists

---

## Best Practices

### ⚠️ Critical Operational Rules

1. **Control Tower APIs require the home region** — `ListEnabledControls`, `EnableControl`, `GetControl` etc. will return `ResourceNotFoundException` if called in any region other than the CT home region. Always determine the home region FIRST.
2. **ListEnabledControls only works on OUs, NOT the Root** — targeting the organization root returns `ValidationException`. Only pass OU ARNs as `targetIdentifier`.
3. **If `aws___run_script` returns "not yet supported" for a service, fall back to `aws___call_aws`** — as of July 2026, `config` is unsupported in run_script. Use `aws configservice ...` via call_aws instead. Do not retry run_script for the same service after a failure.
4. **StackSets must be created in the home region** — the StackSet and all `CreateStackInstances` calls must specify the CT home region.
5. **Config rules are regional** — check each governed region separately for existing rules.
6. **`controlcatalog` ≠ `controltower`** — browsing the controls catalog uses the `controlcatalog` service (in `us-east-1`), NOT `controltower`. There is NO `controltower list-controls` operation. Use `controlcatalog list-controls` or `controlcatalog get-control` to look up available controls. Use `controltower list-enabled-controls` (with full OU ARN, in home region) to check what's already enabled.
7. **Prefer documentation search over catalog pagination** — `aws___search_documentation` is faster and more reliable for finding control identifiers than paginating through `controlcatalog list-controls`.
8. **All resources created by this power MUST use the `kiro-power-` prefix** — StackSets, stack names, SCPs, Config rules, and any other resources. This makes them easily identifiable and filterable. Examples: `kiro-power-ebs-encryption-detective`, `kiro-power-s3-public-block`.

### Control Selection Priority
1. **Always prefer Control Tower catalog controls** — they are AWS-managed, automatically updated, and integrated with Control Tower drift detection
2. **Use SCPs for hard preventive guardrails** — actions that should NEVER be allowed regardless of IAM permissions
3. **Use RCPs for resource-level restrictions** — restrict what can be done TO a resource (e.g., block public access)
4. **Use CloudFormation Guard for proactive validation** — catch issues before resources are created
5. **Use Config Rules for detective monitoring** — continuous compliance checking for existing resources

### SCP Best Practices
- **Check quota before proposing** — SCPs are limited to 10 per OU, RCPs to 5 per OU (hard limits). Always check current count before recommending a new policy.
- Always include a `Condition` to exempt the management account and break-glass roles
- Use `StringNotLike` conditions for flexibility
- Never deny `organizations:*` or `iam:*` broadly — this can lock out the account
- Test in a sandbox OU before applying to production OUs
- Include a description that maps back to the governance intent
- Individual SCP document size limit is 5,120 characters

### StackSet Deployment
- Always deploy from the Control Tower home region
- **Administration role**: Use `AWSControlTowerStackSetRole` in CT environments (not `AWSCloudFormationStackSetAdministrationRole` which may not exist). Verify the role exists before creating the StackSet.
- Use `SERVICE_MANAGED` permission model when deploying to the full Organization
- Use `SELF_MANAGED` with `AWSControlTowerExecution` as execution role when targeting specific OUs
- Set `MaxConcurrentPercentage` to control blast radius
- Set `FailureTolerancePercentage` to prevent cascading failures
- Always deploy stack instances in ALL Control Tower governed regions

### Safety
- **Never deploy without operator confirmation** — always present the full plan first
- **Never modify existing SCPs** — create new ones to maintain auditability
- **Never disable Control Tower controls** — only add new controls
- **Check for conflicts** — ensure new SCPs don't conflict with existing ones
- **Preserve break-glass access** — always exempt emergency access roles from denies

---

## MCP Servers

This power uses the unified AWS MCP Server which provides:

### Tools Used in This Power

| Tool | Purpose in Governance Workflow |
|------|------|
| `aws___call_aws` | Execute AWS API calls (Organizations, Control Tower, CloudFormation, Config, IAM) |
| `aws___run_script` | Multi-step discovery scripts (list OUs, enumerate controls, check duplicates) |
| `aws___search_documentation` | Find Control Tower control identifiers, SCP syntax, Config rule schemas |
| `aws___retrieve_skill` | Load AWS governance skills and best practices |
| `aws___read_documentation` | Read specific AWS docs pages for control specifications |
| `aws___list_regions` | Enumerate regions for StackSet targeting |
| `aws___get_tasks` | Monitor long-running StackSet operations |

### Required IAM Permissions

The management account credentials need:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OrganizationsRead",
      "Effect": "Allow",
      "Action": [
        "organizations:ListAccounts",
        "organizations:ListOrganizationalUnitsForParent",
        "organizations:ListRoots",
        "organizations:DescribeOrganization",
        "organizations:ListPolicies",
        "organizations:DescribePolicy"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ControlTowerRead",
      "Effect": "Allow",
      "Action": [
        "controltower:ListEnabledControls",
        "controltower:ListControls",
        "controltower:GetControl",
        "controltower:GetLandingZone",
        "controltower:ListLandingZones",
        "controltower:GetEnabledControl"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ControlTowerWrite",
      "Effect": "Allow",
      "Action": [
        "controltower:EnableControl",
        "controltower:GetControlOperation"
      ],
      "Resource": "*"
    },
    {
      "Sid": "StackSets",
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStackSet",
        "cloudformation:CreateStackInstances",
        "cloudformation:DescribeStackSet",
        "cloudformation:DescribeStackSetOperation",
        "cloudformation:ListStackSets",
        "cloudformation:ListStackInstances",
        "cloudformation:UpdateStackSet"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ConfigRead",
      "Effect": "Allow",
      "Action": [
        "config:DescribeConfigRules",
        "config:DescribeComplianceByConfigRule",
        "config:GetComplianceDetailsByConfigRule"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SCPManagement",
      "Effect": "Allow",
      "Action": [
        "organizations:CreatePolicy",
        "organizations:AttachPolicy",
        "organizations:ListTargetsForPolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Security Considerations

⚠️ **This power executes write operations against your AWS Organization.** It can:
- Enable Control Tower controls on OUs
- Create and attach SCPs/RCPs
- Deploy CloudFormation StackSets across multiple accounts and regions

**All destructive operations require explicit operator approval before execution.**

The power uses the management account which has broad permissions. Ensure:
- Credentials are short-lived (use `aws login` or SSO)
- Only authorized operators use this power
- Review the deployment plan carefully before approving

## License

This power integrates with the [AWS MCP Server](https://github.com/awslabs/mcp) (Apache-2.0 license).
All steering files and power configuration are licensed under Apache-2.0.

- [Privacy Policy](https://aws.amazon.com/privacy/)
- [Support](https://github.com/kirodotdev/Kiro/issues/new/choose)

---

**Source:** AWS
**License:** Apache-2.0
**Documentation:** https://github.com/awslabs/mcp
