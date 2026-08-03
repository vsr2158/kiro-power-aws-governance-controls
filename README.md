# AWS Governance Control Automation

A [Kiro Power](https://kiro.dev/powers/) that automates AWS governance control deployment across multi-account Organizations — discovers environments, recommends Control Tower catalog controls, authors custom SCPs/Config rules/Guard policies, and deploys via StackSets.

## What it does

Describe your governance intent in natural language. The agent handles the rest:

```
"I need to detect unencrypted EBS volumes on my Prod OU"
```

The power will:
1. **Discover** your AWS Organizations environment, Control Tower setup, and existing controls
2. **Search** the Control Tower catalog (~400 controls) and AWS Config managed rules (~300) for a match
3. **Author** custom controls (SCPs, Config rules, Guard policies) if no catalog match exists
4. **Deploy** via Control Tower API or CloudFormation StackSets — only after your explicit approval
5. **Recommend** complementary controls for defense-in-depth

## Workflow

```
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE 0: Environment Overview (automatic)                           │
│  • Discover CT home region, governed regions                         │
│  • Map all OUs, accounts, existing controls                          │
│  • Display summary table to operator                                 │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE 1: Gather Intent                                              │
│  • What resource/property? (e.g., EBS encryption)                    │
│  • Which OU? (from table above)                                      │
│  • Detective / Preventive / Both?                                    │
│  ⚠️  Blocks until all inputs confirmed                               │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE 2: Scoped Discovery (target OU only)                          │
│  • Check existing CT controls on target OU                           │
│  • Check existing SCPs, Config rules                                 │
│  • Duplicate detection → STOP if already covered                     │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE 3: Control Selection (3-tier fallback)                        │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │ Tier 1: Control Tower Catalog (~400 AWS-managed controls)   │     │
│  │         → If match: EnableControl API                       │     │
│  └─────────────────────────┬───────────────────────────────────┘     │
│                        No match                                       │
│                             ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │ Tier 2: AWS Config Managed Rules (~300 rules)               │     │
│  │         → If match: Deploy via StackSet (no custom code)    │     │
│  └─────────────────────────┬───────────────────────────────────┘     │
│                        No match                                       │
│                             ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │ Tier 3: Custom Control Authoring                            │     │
│  │         → Detective: Guard-based Config rule                │     │
│  │         → Preventive: SCP with exemptions                   │     │
│  │         → Proactive: CloudFormation Guard hook               │     │
│  │         → Always creates ≥2 controls (defense-in-depth)     │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE 5: Deployment (requires explicit "yes")                       │
│                                                                      │
│  • Present full deployment plan (controls, scope, regions, impact)   │
│  • CT Catalog → controltower:EnableControl                           │
│  • SCPs → organizations:CreatePolicy + AttachPolicy                  │
│  • Config/Guard → StackSet to all governed regions                   │
│  • Monitor until SUCCEEDED, verify stack instances                   │
│                                                                      │
│  Safety hooks enforce:                                                │
│  ✓ SCP validation (no unsafe denies, exemptions present)             │
│  ✓ Region enforcement (StackSets use CT home region)                 │
│  ✓ Approval gate (blocks writes without explicit "yes")              │
│  ✓ Post-deploy verification (status report + rollback commands)      │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PHASE 6: Defense-in-Depth Recommendation                            │
│                                                                      │
│  • Detective deployed? → Recommend Preventive or Proactive           │
│  • Preventive deployed? → Recommend Detective (find existing gaps)   │
│  • Proactive deployed? → Recommend Detective + Preventive            │
│  • Search CT catalog first, custom only if no match                  │
└──────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- AWS CLI 2.32.0+
- Management account credentials with Control Tower and Organizations access
- Active Control Tower landing zone
- [Kiro IDE](https://kiro.dev/ide/)

## Installation

### Kiro IDE
Search for **"AWS Governance Control Automation"** in the Powers panel and click Install.

### Manual
Clone this repo into your project and the power will be automatically detected:

```bash
git clone https://github.com/vsr2158/kiro-power-aws-governance-controls.git .kiro/powers/aws-governance-controls
```

## Quick Test

After installation and credential setup, try:

> *"Show me my organization structure and existing governance controls"*

or

> *"I need to detect unencrypted EBS volumes on my Prod OU"*

## Control Types Supported

| Type | Mechanism | Use Case |
|------|-----------|----------|
| **Preventive** | SCP / RCP | Block non-compliant actions org-wide |
| **Detective** | AWS Config Rule | Continuously monitor existing resources |
| **Proactive** | CloudFormation Guard | Validate before deployment |

## Examples

See the included example templates:
- [`kiro-power-ebs-encryption-tagged-prod-detective.yaml`](kiro-power-ebs-encryption-tagged-prod-detective.yaml) — Guard-based Config rule for unencrypted EBS volumes in prod
- [`ec2-sg-restricted-ports-detective-control.yaml`](ec2-sg-restricted-ports-detective-control.yaml) — Guard-based Config rule for open security group ports

## Safety

- **All write operations require explicit approval** before execution
- Resources are prefixed with `kiro-power-` for easy identification
- Break-glass roles are always exempt from deny policies
- Existing controls are never modified or disabled

Four safety hooks run automatically on every AWS API call:

| Hook | Trigger | What it does |
|------|---------|--------------|
| `validate-scp-before-deploy` | PreToolUse | Blocks unsafe SCPs (broad denies, missing exemptions) |
| `validate-stackset-region` | PreToolUse | Ensures StackSets deploy from CT home region only |
| `require-deployment-approval` | PreToolUse | Blocks any write operation without explicit operator "yes" |
| `post-deployment-verification` | PostToolUse | Reports deployment status and provides rollback commands |

## Documentation

- [POWER.md](POWER.md) — Full power documentation, onboarding, and best practices
- [steering/](steering/) — Agent steering files for each workflow phase

## License

Apache-2.0

- [Privacy Policy](https://aws.amazon.com/privacy/)
- [Support](https://github.com/kirodotdev/Kiro/issues/new/choose)
