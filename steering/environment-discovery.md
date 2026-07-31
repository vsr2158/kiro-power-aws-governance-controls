# Environment Discovery

## Purpose
This steering file guides the agent through discovering the current state of the multi-account environment, including organization structure, existing controls, and potential duplicates.

## ⚠️ CRITICAL: Region Requirements

**Control Tower APIs are REGIONAL.** They MUST be called in the **Control Tower home region**, NOT the default region.

| Service | Region Requirement |
|---------|-------------------|
| `organizations` | Global (any region works, typically `us-east-1`) |
| `controltower` | **MUST use the Control Tower home region** (e.g., `ap-southeast-2`) |
| `config` / `configservice` | Regional — must call in each governed region separately |
| `cloudformation` (StackSets) | **MUST use the Control Tower home region** |

**The agent MUST determine the Control Tower home region FIRST before making any `controltower` API calls.**

### How to Find the Control Tower Home Region

Use `aws___call_aws`:
```
aws controltower list-landing-zones
```

The landing zone ARN contains the home region: `arn:aws:controltower:<HOME_REGION>:<ACCOUNT>:landingzone/<ID>`

If this fails in the default region, try common CT home regions: `us-east-1`, `us-west-2`, `eu-west-1`, `ap-southeast-2`, `ap-southeast-1`.

**Once found, ALL subsequent `controltower` and `cloudformation` (StackSet) calls MUST include `--region <HOME_REGION>`.**

## ⚠️ CRITICAL: Service Support in run_script vs call_aws

The `aws___run_script` tool (sandboxed Python/boto3) does NOT support all AWS services. If a service is unsupported, it returns: `"ToolError: <service>: <service> is not yet supported"`

**Fallback rule**: If `aws___run_script` returns a "not yet supported" error for any service, **immediately switch to `aws___call_aws`** for that service. Do NOT retry with run_script.

### Known unsupported services (as of July 2026)

| Service | `aws___run_script` | `aws___call_aws` | CLI service name |
|---------|---|---|---|
| **Config** | ❌ Not yet supported | ✅ Works | `configservice` |

### Services confirmed working in run_script

| Service | boto3 service_name |
|---------|---|
| Organizations | `organizations` |
| Control Tower | `controltower` |
| Control Catalog | `controlcatalog` |
| CloudFormation | `cloudformation` |
| IAM | `iam` |
| STS | `sts` |

### Fallback pattern

```
1. Try the operation in aws___run_script
2. If you get "not yet supported" error:
   → Switch to aws___call_aws with the correct CLI service name
   → Do NOT retry run_script for that service in this session
3. Remember: CLI service names may differ from boto3 names
   (e.g., boto3 'config' → CLI 'configservice')
```

### Config operations via aws___call_aws

```
aws configservice describe-config-rules --region <REGION>
aws configservice describe-compliance-by-config-rule --region <REGION>
aws configservice get-compliance-details-by-config-rule --config-rule-name <NAME> --region <REGION>
```

## Step 1: Discover Organization Structure

Use `aws___run_script` to gather the full organization topology:

```python
# Discover organization structure
org = await call_boto3(service_name='organizations', operation_name='DescribeOrganization')
org_id = org['Organization']['Id']
master_account_id = org['Organization']['MasterAccountId']

roots = await call_boto3(service_name='organizations', operation_name='ListRoots')
root_id = roots['Roots'][0]['Id']

# Get all OUs recursively
async def get_ous(parent_id):
    ous = await call_boto3(
        service_name='organizations',
        operation_name='ListOrganizationalUnitsForParent',
        params={'ParentId': parent_id}
    )
    result = []
    for ou in ous.get('OrganizationalUnits', []):
        children = await get_ous(ou['Id'])
        result.append({'Id': ou['Id'], 'Name': ou['Name'], 'Children': children})
    return result

ou_tree = await get_ous(root_id)

# Get all accounts
accounts = await call_boto3(service_name='organizations', operation_name='ListAccounts')

result = {
    'org_id': org_id,
    'master_account_id': master_account_id,
    'root_id': root_id,
    'ou_tree': ou_tree,
    'total_accounts': len(accounts.get('Accounts', []))
}
result
```

## Step 2: Identify Control Tower Configuration

**This is the FIRST thing to do before any Control Tower API calls.**

### Step 2a: Find the Control Tower Home Region

Use `aws___call_aws` (NOT run_script) to find the landing zone. Try the default region first:

```
aws controltower list-landing-zones
```

If this returns `ResourceNotFoundException`, it means CT is not in this region. Try other regions:
```
aws controltower list-landing-zones --region ap-southeast-2
aws controltower list-landing-zones --region us-east-1
aws controltower list-landing-zones --region eu-west-1
```

Extract the home region from the landing zone ARN: `arn:aws:controltower:<HOME_REGION>:...`

### Step 2b: Get Landing Zone Details

Once you have the home region, get full details:

```
aws controltower get-landing-zone --landing-zone-identifier <LANDING_ZONE_ARN> --region <HOME_REGION>
```

From the response, extract:
- **Home region** — where Control Tower is deployed
- **Governed regions** — all regions where controls will be applied
- **Status** — must be ACTIVE
- **Landing zone version** — check if it's the latest version
- **Drift status** — check if landing zone is drifted

### Step 2c: Check Landing Zone Version and Drift

```
aws controltower list-landing-zones --region <HOME_REGION>
```

If the landing zone version is outdated or drift is detected, warn the operator:

```
⚠️  LANDING ZONE ATTENTION REQUIRED

- Version: [current] (latest available: [latest])
- Drift Status: [IN_SYNC / DRIFTED]

[If outdated]: Your landing zone is not on the latest version. 
  Some newer controls may not be available until you update.
  Consider running a landing zone update from the Control Tower console.

[If drifted]: Your landing zone has drifted from its expected configuration.
  Controls may not behave as expected. Consider resolving drift before
  enabling new controls.
```

### Step 2d: Check OU Registration and Account Enrollment

**Controls only apply to registered OUs and enrolled accounts.** Unregistered OUs and unenrolled accounts will NOT be governed by any controls you enable.

Check OU registration status:
```
aws controltower list-organizational-units --region <HOME_REGION>
```

For each OU in the target scope, check if it's registered with Control Tower. The response shows registration status.

Check account enrollment:
```
aws controltower list-enabled-baselines --region <HOME_REGION>
```

Or check individual accounts:
```
aws controltower get-enrolled-account --account-id <ACCOUNT_ID> --region <HOME_REGION>
```

**If an OU is NOT registered or accounts are NOT enrolled, warn the operator:**

```
⚠️  ENROLLMENT ISSUE DETECTED

The following OUs/accounts are NOT fully enrolled in Control Tower:

┌──────────────────────────────────────────────────────────┐
│ Name          │ ID                  │ Status             │
├───────────────┼─────────────────────┼────────────────────┤
│ Sandbox OU    │ ou-xxxx-xxxxxxxx    │ NOT REGISTERED     │
│ Account-Dev   │ 444444444444        │ NOT ENROLLED       │
└──────────────────────────────────────────────────────────┘

⚠️  Controls enabled on unregistered OUs will NOT take effect.
    Unenrolled accounts within registered OUs will NOT be governed.

Options:
1. Register the OU / enroll the account first (recommended)
2. Proceed anyway (control won't apply to unenrolled resources)
3. Choose a different target OU that is fully enrolled

Would you like me to help register/enroll, or proceed with a different target?
```

### Step 2e: Check Security Hub and Config Delegated Administrators

Many detective controls are deployed via **Security Hub CSPM** or **AWS Config Organization rules**. Check delegated admins for both.

**Check delegated admin for Security Hub:**
```
aws organizations list-delegated-administrators --service-principal securityhub.amazonaws.com
```

**Check delegated admin for Config:**
```
aws organizations list-delegated-administrators --service-principal config-multiaccountsetup.amazonaws.com
```

If a Config delegated admin exists, Organization Config Rules MUST be deployed from that account, not the management account.

**Check if Security Hub is enabled in the management account:**
```
aws securityhub describe-hub --region <HOME_REGION>
```

**Check for the Control Tower Security Hub standard:**
```
aws securityhub get-enabled-standards --region <HOME_REGION>
```

Look for the standard ARN containing `service-managed-aws-control-tower`.

**Include in the environment summary:**

```
Security Hub:
- Delegated Admin: [account-id / name] or ⚠️ NOT CONFIGURED
- CT CSPM Standard: [✓ Enabled / ⚠️ Not enabled]

AWS Config:
- Delegated Admin: [account-id / name] or "Management account (no delegation)"
- Org Config Rules: Deploy from [delegated admin / management account]
```

**If no delegated admin is configured, warn:**

```
⚠️  SECURITY HUB DELEGATED ADMIN NOT CONFIGURED

Security Hub does not have a delegated administrator account.
This means:
- Security Hub findings are only visible in the management account
- Cross-account security posture aggregation is not active
- Some Control Tower detective controls rely on Security Hub CSPM

Recommendation: Delegate administration to your Security/Audit account.
Would you like guidance on setting this up?
```

**If CSPM standard is not enabled, warn:**

```
⚠️  SECURITY HUB CSPM STANDARD NOT ENABLED

The "Service-Managed Standard: AWS Control Tower" is not active.
Controls deployed by Control Tower that rely on Security Hub CSPM
(prefixed securityhub-*) will not generate findings.

This is typically enabled automatically when Control Tower enables
detective controls. If missing, it may indicate a configuration issue.
```

### Step 2f: Store Home Region for All Subsequent Calls

**From this point forward:**
- ALL `controltower` API calls → include `--region <HOME_REGION>` or `region_name='<HOME_REGION>'`
- ALL `cloudformation` StackSet calls → include `--region <HOME_REGION>` or `region_name='<HOME_REGION>'`
- ALL `configservice` calls → must specify each governed region individually

## Step 3: List Enabled Control Tower Controls

**IMPORTANT**: 
- Must use the **home region** for all `controltower` API calls
- `ListEnabledControls` only works on **OU targets**, NOT on the Root — targeting the Root returns `ValidationException`
- Use the **OU ARN** format: `arn:aws:organizations::<ACCOUNT>:ou/<ORG_ID>/<OU_ID>`

### ⚠️ Searching the Control Tower Controls Catalog

**There is NO `controltower list-controls` API.** The catalog is in a SEPARATE service: `controlcatalog`.

**Correct APIs for browsing the catalog:**

| What you want | Service | Operation | Region |
|---|---|---|---|
| List enabled controls on an OU | `controltower` | `list-enabled-controls` | **CT home region** |
| Search available controls in catalog | `controlcatalog` | `list-controls` | `us-east-1` (global) |
| Get control details by ARN | `controlcatalog` | `get-control` | `us-east-1` (global) |

**`controlcatalog` is a GLOBAL service — always call it in `us-east-1`.**

### How to Find a Control in the Catalog

**Method 1: Use `aws___search_documentation`** (PREFERRED — fastest)
```
Search: "Control Tower [detective|preventive|proactive] control [resource] [property]"
Topics: ["reference_documentation"]
```
This often returns the control identifier, alias, and implementation details directly.

**Method 2: Use `controlcatalog list-controls` with JMESPath filtering**
```
aws controlcatalog list-controls --region us-east-1 --query "Controls[?contains(Name, 'keyword')]"
```

⚠️ **Known limitations of `controlcatalog list-controls`:**
- Server-side filtering is limited — the `--query` JMESPath filter only applies to the CURRENT PAGE
- The API paginates (default ~100 per page) — a filter on page 1 may return empty while the match is on page 5
- **Do NOT rely solely on JMESPath filtering** — if you get empty results, it doesn't mean the control doesn't exist

**Method 3: Use known global IDs directly**

Common detective controls and their global IDs:

| Control | Global ID | Alias |
|---|---|---|
| Detect EBS volumes unencrypted | `503uicglhjkokaajywfpt6ros` | AWS-GR_ENCRYPTED_VOLUMES |
| Detect public RDS instances | (search docs) | AWS-GR_RDS_INSTANCE_PUBLIC_ACCESS_CHECK |
| Detect S3 public read | (search docs) | AWS-GR_S3_BUCKET_PUBLIC_READ_PROHIBITED |
| Detect S3 public write | (search docs) | AWS-GR_S3_BUCKET_PUBLIC_WRITE_PROHIBITED |
| Detect root MFA disabled | (search docs) | AWS-GR_ROOT_ACCOUNT_MFA_ENABLED |
| Detect unrestricted SSH | (search docs) | AWS-GR_RESTRICTED_SSH |

**Method 4: If you know the alias (e.g., `AWS-GR_ENCRYPTED_VOLUMES`), construct the identifier:**
- For legacy CT controls: `arn:aws:controltower:<HOME_REGION>::control/<ALIAS>`
- For catalog controls: `arn:aws:controlcatalog:::control/<GLOBAL_ID>`

### Recommended Search Strategy

1. **First**: Use `aws___search_documentation` to find the control name, identifier, and global ID
2. **Then**: Use the global ID with `EnableControl` — no need to enumerate the full catalog
3. **Only if docs don't have it**: Fall back to `controlcatalog list-controls` with pagination

### Using `controlcatalog list-controls` correctly

```
# Search by keyword — NOTE: only filters current page!
aws controlcatalog list-controls --region us-east-1 \
  --query "Controls[?contains(Name, 'EBS') && Behavior=='DETECTIVE']"

# If empty result, paginate:
aws controlcatalog list-controls --region us-east-1 \
  --starting-token <pagination_token> \
  --query "Controls[?contains(Name, 'EBS')]"

# Get details of a specific control by ARN:
aws controlcatalog get-control --region us-east-1 \
  --control-arn "arn:aws:controlcatalog:::control/503uicglhjkokaajywfpt6ros"
```

---

Use `aws___run_script` with **explicit `region_name`**:

```python
# List enabled controls for each OU (NOT the root!)
# HOME_REGION must be set from Step 2
home_region = 'ap-southeast-2'  # Replace with actual CT home region

ou_arns = [f'arn:aws:organizations::{mgmt_account_id}:ou/{org_id}/{ou_id}' for ou_id in ou_ids]

# Do NOT include the root — CT controls are not applied to root
enabled_results = await asyncio.gather(*[
    call_boto3(
        service_name='controltower',
        operation_name='ListEnabledControls',
        region_name=home_region,  # ← CRITICAL: must be CT home region
        params={'targetIdentifier': ou_arn}
    ) for ou_arn in ou_arns
], return_exceptions=True)

enabled_controls = []
for ou_arn, result in zip(ou_arns, enabled_results):
    if isinstance(result, dict):
        for ctrl in result.get('enabledControls', []):
            enabled_controls.append({
                'controlIdentifier': ctrl['controlIdentifier'],
                'targetOU': ou_arn,
                'status': ctrl.get('statusSummary', {}).get('status')
            })

result = {
    'total_enabled_controls': len(enabled_controls),
    'controls': enabled_controls
}
result
```

### Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `ResourceNotFoundException: must create a landing zone first` | Wrong region | Use CT home region |
| `ValidationException: controls are not applied to the Root` | Targeted root instead of OU | Only target OU ARNs, not root ARN |
| `AccessDeniedException` | Missing permissions | Ensure `controltower:ListEnabledControls` permission |

## Step 4: List Existing SCPs

```python
# List all SCPs in the organization
policies = await call_boto3(
    service_name='organizations',
    operation_name='ListPolicies',
    params={'Filter': 'SERVICE_CONTROL_POLICY'}
)

scp_details = []
for policy in policies.get('Policies', []):
    detail = await call_boto3(
        service_name='organizations',
        operation_name='DescribePolicy',
        params={'PolicyId': policy['Id']}
    )
    targets = await call_boto3(
        service_name='organizations',
        operation_name='ListTargetsForPolicy',
        params={'PolicyId': policy['Id']}
    )
    scp_details.append({
        'id': policy['Id'],
        'name': policy['Name'],
        'description': policy.get('Description', ''),
        'content': detail['Policy']['Content'],
        'targets': [t['Name'] for t in targets.get('Targets', [])]
    })

result = {'scps': scp_details, 'total': len(scp_details)}
result
```

## Step 5: List Existing Config Rules

**⚠️ `config`/`configservice` is NOT supported in `aws___run_script`.** Use `aws___call_aws` instead.

Check for existing Config rules in EACH governed region separately:

```
aws configservice describe-config-rules --region us-east-1
aws configservice describe-config-rules --region ap-southeast-2
aws configservice describe-config-rules --region us-west-2
```

For each region, look for rules matching the operator's intent. Key fields:
- `ConfigRuleName` — human-readable name
- `Source.Owner` — `AWS` (managed) or `CUSTOM_LAMBDA` / `CUSTOM_POLICY`
- `Source.SourceIdentifier` — the managed rule identifier (e.g., `ENCRYPTED_VOLUMES`, `RDS_INSTANCE_PUBLIC_ACCESS_CHECK`)
- `Scope.ComplianceResourceTypes` — what resource types the rule evaluates
- `CreatedBy` — indicates if deployed by Security Hub, Control Tower, or custom

### Filtering for Relevant Rules

When checking for duplicates of the operator's intent, filter by:
1. **SourceIdentifier** matching the expected managed rule (e.g., `ENCRYPTED_VOLUMES` for EBS encryption)
2. **ConfigRuleName** containing relevant keywords (e.g., `volume`, `ebs`, `encrypt`)
3. **CreatedBy** — rules created by `securityhub.amazonaws.com` are Security Hub managed; rules from `controltower.amazonaws.com` are CT managed

### Example: Check for EBS encryption rules across regions

```
aws configservice describe-config-rules --region ap-southeast-2 --query "ConfigRules[?Source.SourceIdentifier=='ENCRYPTED_VOLUMES']"
aws configservice describe-config-rules --region us-east-1 --query "ConfigRules[?Source.SourceIdentifier=='ENCRYPTED_VOLUMES']"
```

## Step 6: Duplicate Detection

After gathering all existing controls, check for duplicates against the operator's intent:

### Matching Logic

1. **Control Tower controls**: Match by control identifier description keywords
2. **SCPs**: Parse the policy JSON and check if `Action` + `Resource` + `Condition` already covers the intent
3. **Config rules**: Match by `SourceIdentifier` (for managed rules) or by rule name pattern

### Report Duplicates

If a duplicate is found, format:

```
⚠️  EXISTING CONTROL DETECTED

Your request: "[operator's intent]"

Existing control found:
- Type: [SCP / Config Rule / CT Control]
- Name: [name]
- Scope: [where it's applied]
- Coverage: [Full / Partial]

[If full coverage]: This control already covers your requirement. No additional deployment needed.
[If partial coverage]: This control partially covers your requirement. I can extend it or create a complementary control.

What would you like to do?
1. Keep existing control (no changes)
2. Extend scope of existing control
3. Create an additional complementary control
```

## Environment Summary Template

After discovery, present this summary **including the full OU and account structure** so the operator can select their target OU:

```
═══════════════════════════════════════════════════════════
ENVIRONMENT SUMMARY
═══════════════════════════════════════════════════════════

Organization: [org-id]
Management Account: [account-id]
Control Tower Home Region: [region]
Governed Regions: [region-1, region-2, ...]
Landing Zone Version: [version] [✓ latest / ⚠️ update available]
Landing Zone Status: [ACTIVE] [✓ in sync / ⚠️ DRIFTED]

Security Hub:
- Delegated Admin: [account-id (name)] or ⚠️ Not configured
- CT CSPM Standard: [✓ Enabled / ⚠️ Not enabled]

AWS Config:
- Delegated Admin: [account-id (name)] or Management account (no delegation)
- Org Config Rules deploy from: [account]

Organizational Units:
┌──────────────────────────────────────────────────────────────────────┐
│ # │ OU Name        │ OU ID              │ Accounts │ Controls │ CT Status    │
├───┼────────────────┼────────────────────┼──────────┼──────────┼──────────────┤
│ 1 │ Security       │ ou-xxxx-xxxxxxxx   │ 2        │ 12       │ ✓ Registered │
│ 2 │ Production     │ ou-xxxx-xxxxxxxx   │ 3        │ 8        │ ✓ Registered │
│ 3 │ Development    │ ou-xxxx-xxxxxxxx   │ 2        │ 5        │ ✓ Registered │
│ 4 │ Sandbox        │ ou-xxxx-xxxxxxxx   │ 1        │ 0        │ ⚠️ Not Reg.  │
└──────────────────────────────────────────────────────────────────────┘

Accounts:
┌──────────────────────────────────────────────────────────────────────┐
│ Account ID   │ Name              │ OU            │ Status │ Enrolled │
├──────────────┼───────────────────┼───────────────┼────────┼──────────┤
│ 111111111111 │ Security-Audit    │ Security      │ ACTIVE │ ✓ Yes    │
│ 222222222222 │ Security-Log      │ Security      │ ACTIVE │ ✓ Yes    │
│ 333333333333 │ Prod-App          │ Production    │ ACTIVE │ ✓ Yes    │
│ 444444444444 │ Dev-App           │ Development   │ ACTIVE │ ⚠️ No    │
│ 555555555555 │ Sandbox-Test      │ Sandbox       │ ACTIVE │ ⚠️ No    │
└──────────────────────────────────────────────────────────────────────┘

[If any issues]:
⚠️  ATTENTION: [X] OU(s) not registered, [Y] account(s) not enrolled.
    Controls will NOT apply to unregistered/unenrolled resources.

═══════════════════════════════════════════════════════════
```

**IMMEDIATELY after showing this summary, ask the operator for their target OU:**

```
Which OU(s) should the control be applied to? (refer by number or name above)

💡 Best practice: Test in a sandbox/dev OU first, then roll out to production.
```
