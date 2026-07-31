# StackSet Deployment

## Purpose
This steering file guides the agent through deploying governance controls via CloudFormation StackSets from the Control Tower home region.

## Deployment Principles

1. **Always deploy from Control Tower home region** — this ensures consistency with Control Tower's own deployments
2. **Always target ALL governed regions** — controls must be active everywhere Control Tower manages
3. **Always use `AWSControlTowerExecution` role** — for stack instances in member accounts
4. **Always present the plan before deploying** — operator must explicitly approve
5. **SCPs are deployed separately** — they go through Organizations API, not StackSets

---

## Deployment Decision Tree

```
Is this a Control Tower catalog control?
├── YES → Use controltower:EnableControl API
│         (no StackSet needed, CT manages deployment)
└── NO → Is this an SCP/RCP?
    ├── YES → Deploy via Organizations API (CreatePolicy + AttachPolicy)
    │         (SCPs/RCPs are org-level, not per-account)
    └── NO → Deploy via CloudFormation StackSet
              (Config rules, Guard hooks, Lambda-based rules)
```

---

## Deploying Control Tower Catalog Controls

**CRITICAL: Must use the Control Tower home region for ALL controltower API calls.**

```python
# Enable a Control Tower control on a target OU
# HOME_REGION must be determined from Step 2 of environment-discovery
result = await call_boto3(
    service_name='controltower',
    operation_name='EnableControl',
    region_name=home_region,  # ← CRITICAL: CT home region, NOT default
    params={
        'controlIdentifier': 'arn:aws:controltower:<HOME_REGION>::control/[CONTROL_ID]',
        'targetIdentifier': 'arn:aws:organizations::[MGMT_ACCOUNT]:ou/[ORG_ID]/[OU_ID]'
    }
)

# Monitor the operation — also in home region
operation_id = result['operationIdentifier']
status = await call_boto3(
    service_name='controltower',
    operation_name='GetControlOperation',
    region_name=home_region,  # ← CRITICAL: same home region
    params={'operationIdentifier': operation_id}
)
```

**Common mistake**: The control identifier ARN also contains the region. Ensure it matches the home region:
- ✅ `arn:aws:controltower:ap-southeast-2::control/AWS-GR_ENCRYPTED_VOLUMES`
- ❌ `arn:aws:controltower:us-east-1::control/AWS-GR_ENCRYPTED_VOLUMES` (wrong region for this org)

**Wait for completion** — Control Tower operations can take several minutes. Poll `GetControlOperation` until status is `SUCCEEDED` or `FAILED`.

---

## Deploying SCPs via Organizations API

SCPs are organization-level policies, NOT deployed via StackSets.

```python
# Step 1: Create the SCP
create_result = await call_boto3(
    service_name='organizations',
    operation_name='CreatePolicy',
    params={
        'Content': json.dumps(scp_document),
        'Description': f'Governance control: {governance_intent}',
        'Name': f'kiro-power-{control_name}',
        'Type': 'SERVICE_CONTROL_POLICY'
    }
)
policy_id = create_result['Policy']['PolicySummary']['Id']

# Step 2: Attach to target OU(s)
for ou_id in target_ou_ids:
    await call_boto3(
        service_name='organizations',
        operation_name='AttachPolicy',
        params={
            'PolicyId': policy_id,
            'TargetId': ou_id
        }
    )
```

### SCP Limit Check

Before creating/attaching an SCP, verify limits:

```python
# Check how many SCPs are attached to the target
for ou_id in target_ou_ids:
    policies = await call_boto3(
        service_name='organizations',
        operation_name='ListPoliciesForTarget',
        params={
            'TargetId': ou_id,
            'Filter': 'SERVICE_CONTROL_POLICY'
        }
    )
    count = len(policies.get('Policies', []))
    if count >= 10:
        # STOP - cannot attach more SCPs to this target (hard limit: 10)
        # Inform operator and suggest consolidating existing SCPs
        pass
    elif count >= 8:
        # WARN - nearing quota, suggest consolidation as follow-up
        pass
```

---

## Deploying via CloudFormation StackSets

### Step 1: Determine Deployment Parameters

```python
# Get Control Tower governed regions
lz = await call_boto3(
    service_name='controltower',
    operation_name='GetLandingZone',
    region_name=home_region,
    params={'landingZoneIdentifier': landing_zone_arn}
)

# Home region is the region field of the landing zone ARN (always present for this
# regional resource, e.g. arn:aws:controltower:<HOME_REGION>:<acct>:landingzone/<id>).
# Do NOT use governedRegions[0] — the list order does not guarantee the home region is first.
home_region = landing_zone_arn.split(':')[3]
governed_regions = lz['landingZone']['manifest']['governedRegions']

# Get target OU IDs
target_ous = [ou_id for ou_id in selected_target_ous]
```

### Step 2: Create the StackSet

**IMPORTANT**: This API call MUST be made in the Control Tower home region.

**Naming convention**: All StackSets and stacks created by this power MUST use the prefix `kiro-power-*`.

**Administration Role**: In a Control Tower environment, use `AWSControlTowerStackSetRole` (not the generic `AWSCloudFormationStackSetAdministrationRole` which may not exist). The agent should verify which role is available before creating the StackSet.

```python
# Determine the correct administration role
# Control Tower environments use AWSControlTowerStackSetRole
admin_role_candidates = [
    f'arn:aws:iam::{management_account_id}:role/service-role/AWSControlTowerStackSetRole',
    f'arn:aws:iam::{management_account_id}:role/AWSCloudFormationStackSetAdministrationRole'
]

# Check which role exists (prefer CT role in CT environments)
for role_arn in admin_role_candidates:
    try:
        role_name = role_arn.split('/')[-1]
        await call_boto3(
            service_name='iam',
            operation_name='GetRole',
            params={'RoleName': role_name}
        )
        admin_role_arn = role_arn
        break
    except Exception:
        continue

# Create StackSet in the home region
stackset_name = f'kiro-power-{control_name}'

stackset_result = await call_boto3(
    service_name='cloudformation',
    operation_name='CreateStackSet',
    region_name=home_region,
    params={
        'StackSetName': stackset_name,
        'Description': f'Governance control: {governance_intent}',
        'TemplateBody': cloudformation_template_yaml,
        'Parameters': [
            {
                'ParameterKey': 'GovernanceControlName',
                'ParameterValue': f'kiro-power-{control_name}'
            }
        ],
        'Capabilities': ['CAPABILITY_NAMED_IAM'],
        'PermissionModel': 'SELF_MANAGED',
        'AdministrationRoleARN': admin_role_arn,
        'ExecutionRoleName': 'AWSControlTowerExecution',
        'Tags': [
            {'Key': 'ManagedBy', 'Value': 'kiro-power-governance-controls'},
            {'Key': 'GovernanceIntent', 'Value': governance_intent[:256]},
            {'Key': 'CreatedDate', 'Value': datetime.datetime.now().isoformat()}
        ]
    }
)
```

### Step 3: Create Stack Instances

Deploy to ALL governed regions across target accounts:

```python
# Get all account IDs in target OUs
account_ids = []
for ou_id in target_ous:
    accounts = await call_boto3(
        service_name='organizations',
        operation_name='ListAccountsForParent',
        params={'ParentId': ou_id}
    )
    account_ids.extend([a['Id'] for a in accounts.get('Accounts', []) if a['Status'] == 'ACTIVE'])

# Remove management account from targets (controls don't apply there)
account_ids = [a for a in account_ids if a != management_account_id]

# Create stack instances in ALL governed regions
instance_result = await call_boto3(
    service_name='cloudformation',
    operation_name='CreateStackInstances',
    region_name=home_region,
    params={
        'StackSetName': f'kiro-power-{control_name}',
        'Accounts': account_ids,
        'Regions': governed_regions,
        'OperationPreferences': {
            'RegionConcurrencyType': 'PARALLEL',
            'MaxConcurrentPercentage': 10,
            'FailureTolerancePercentage': 0
        }
    }
)

operation_id = instance_result['OperationId']
```

### Step 4: Monitor Deployment

```python
# Check the operation status ONCE. StackSet operations can run for many minutes,
# so do NOT block-poll inside run_script (a long loop will hit the sandbox execution cap).
# If the status is still running, re-run this check on a later turn until it is terminal.
status = await call_boto3(
    service_name='cloudformation',
    operation_name='DescribeStackSetOperation',
    region_name=home_region,
    params={
        'StackSetName': f'kiro-power-{control_name}',
        'OperationId': operation_id
    }
)

op_status = status['StackSetOperation']['Status']
# Terminal states: SUCCEEDED, FAILED, STOPPED.
# Non-terminal (e.g. RUNNING, STOPPING) -> poll again on a later turn.
result = {'op_status': op_status}
result
```

---

## Post-Deployment Verification

After successful deployment:

```python
# Verify stack instances are in CURRENT state
instances = await call_boto3(
    service_name='cloudformation',
    operation_name='ListStackInstances',
    region_name=home_region,
    params={
        'StackSetName': f'kiro-power-{control_name}'
    }
)

summary = {
    'total': len(instances.get('Summaries', [])),
    'current': sum(1 for i in instances.get('Summaries', []) if i['Status'] == 'CURRENT'),
    'failed': [i for i in instances.get('Summaries', []) if i['Status'] != 'CURRENT']
}
```

---

## Deployment Report Template

After deployment completes, present:

```
═══════════════════════════════════════════════════════════
DEPLOYMENT COMPLETE
═══════════════════════════════════════════════════════════

Governance Intent: [original request]

Deployed Controls:
✓ [Control 1 type]: [name] — [status]
✓ [Control 2 type]: [name] — [status]

StackSet: kiro-power-[name]
- Region: [home region]
- Stack Instances: [X] successful / [Y] total
- Regions Covered: [list]
- Accounts Covered: [count]

[If SCP was attached]:
SCP: kiro-power-[name]
- Policy ID: [id]
- Attached to: [OU names]

Next Steps:
1. Verify compliance: Check Config rule results in ~15 minutes
2. Test preventive control: Attempt to create a non-compliant resource
3. Monitor: Review CloudWatch metrics for control evaluations

⚠️  If issues arise:
- StackSet: aws cloudformation delete-stack-instances / delete-stack-set
- SCP: aws organizations detach-policy / delete-policy
- CT Control: aws controltower disable-control
═══════════════════════════════════════════════════════════
```

---

## Rollback Procedure

If the operator needs to rollback:

### Rollback StackSet
```python
# Delete stack instances first
await call_boto3(
    service_name='cloudformation',
    operation_name='DeleteStackInstances',
    region_name=home_region,
    params={
        'StackSetName': stackset_name,
        'Accounts': account_ids,
        'Regions': governed_regions,
        'RetainStacks': False
    }
)

# Then delete the StackSet itself
await call_boto3(
    service_name='cloudformation',
    operation_name='DeleteStackSet',
    region_name=home_region,
    params={'StackSetName': stackset_name}
)
```

### Rollback SCP
```python
# Detach from targets first
for ou_id in target_ous:
    await call_boto3(
        service_name='organizations',
        operation_name='DetachPolicy',
        params={'PolicyId': policy_id, 'TargetId': ou_id}
    )

# Then delete
await call_boto3(
    service_name='organizations',
    operation_name='DeletePolicy',
    params={'PolicyId': policy_id}
)
```

**IMPORTANT**: Always confirm rollback with the operator before executing. Removing controls in production can expose the organization to compliance gaps.
