# Custom Control Authoring

## Purpose
This steering file guides the agent through creating custom governance controls when Control Tower catalog controls don't provide sufficient coverage.

## Control Types and When to Use Them

| Control Type | Mechanism | When to Use |
|---|---|---|
| **Preventive (SCP)** | Service Control Policy | Block API actions org-wide regardless of IAM permissions |
| **Preventive (RCP)** | Resource Control Policy | Restrict what can be done TO a resource (e.g., block public access) |
| **Proactive (Guard)** | CloudFormation Guard | Validate resource properties in CloudFormation templates before deployment |
| **Detective (Config)** | AWS Config Rule | Continuously evaluate existing resources for compliance |

## Defense-in-Depth Strategy

**Always create at least TWO controls** for comprehensive coverage:

1. A **detective** control (Config rule) to find existing non-compliant resources
2. A **preventive** or **proactive** control to block future non-compliance

This ensures:
- Existing violations are discovered and can be remediated
- New violations are prevented at creation time

---

## Authoring SCPs (Service Control Policies)

### SCP Template

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonCompliant[ResourceType][Property]",
      "Effect": "Deny",
      "Action": [
        "[service]:[CreateAction]",
        "[service]:[ModifyAction]"
      ],
      "Resource": "*",
      "Condition": {
        "[ConditionOperator]": {
          "[ConditionKey]": "[non-compliant-value]"
        }
      }
    }
  ]
}
```

### SCP Mandatory Requirements

1. **Always include exemptions** for break-glass and automation roles:
```json
"Condition": {
  "StringNotLike": {
    "aws:PrincipalARN": [
      "arn:aws:iam::*:role/AWSControlTowerExecution",
      "arn:aws:iam::*:role/OrganizationAccountAccessRole",
      "arn:aws:iam::*:role/BreakGlassRole"
    ]
  }
}
```

2. **Never deny these actions** (will lock out the account):
   - `organizations:*`
   - `account:*`
   - `iam:CreateServiceLinkedRole` (needed by many AWS services)
   - `support:*`

3. **Use specific actions** — never use wildcards like `s3:*`

4. **Always add a descriptive Sid** that maps to the governance intent

### SCP Examples

**Amazon S3 encryption — note on scope:**

> S3 **bucket default encryption cannot be enforced with an SCP.** There is no IAM
> condition key for a bucket's encryption, and encryption is not set at `CreateBucket`
> (it is a separate `PutBucketEncryption` call). Blocking "unencrypted bucket creation"
> is also unnecessary: since Jan 5, 2023, S3 automatically applies SSE-S3 as a baseline
> to every bucket, so an unencrypted bucket cannot exist.
> (See the [Default encryption FAQ](https://docs.aws.amazon.com/AmazonS3/latest/userguide/default-encryption-faq.html).)
>
> - To require **KMS** default bucket encryption: use a proactive control
>   (CloudFormation Guard / `CT.S3.PR.10`) plus the detective Config rule
>   `S3_DEFAULT_ENCRYPTION_KMS`.
> - SCPs enforce encryption only at the **object** layer (`s3:PutObject`), shown below.
>   (See [Bucket policy examples using condition keys](https://docs.aws.amazon.com/AmazonS3/latest/userguide/amazon-s3-policy-keys.html).)

**Require SSE-KMS on S3 object uploads (SCP-enforceable):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonKmsObjectUploads",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        },
        "StringNotLike": {
          "aws:PrincipalARN": [
            "arn:aws:iam::*:role/AWSControlTowerExecution"
          ]
        }
      }
    }
  ]
}
```

Note: this is an SCP, so there is no `Principal` element. AWS's published *bucket
policy* examples for the same goal include `"Principal": "*"` — that element applies
to resource-based policies (bucket policies), not SCPs.

**Block public RDS instances:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicRDSInstances",
      "Effect": "Deny",
      "Action": [
        "rds:CreateDBInstance",
        "rds:ModifyDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "rds:PubliclyAccessible": "true"
        },
        "StringNotLike": {
          "aws:PrincipalARN": [
            "arn:aws:iam::*:role/AWSControlTowerExecution"
          ]
        }
      }
    }
  ]
}
```

---

## Authoring RCPs (Resource Control Policies)

### When to Use RCPs vs SCPs

- **SCP**: Controls what *principals* in your org can DO (action-based)
- **RCP**: Controls what can be done *TO resources* in your org (resource-based)

RCPs are ideal for:
- Blocking cross-account access to resources
- Preventing public access to resources
- Restricting resource sharing outside the organization

### RCP Template

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictResourceAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "[service]:[action]",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "${aws:ResourceOrgID}"
        }
      }
    }
  ]
}
```

---

## Authoring CloudFormation Guard Rules (Proactive)

### Guard Rule Template

```ruby
# Rule: [description of what's being validated]
# Intent: [operator's governance intent]

let [resource_type] = Resources.*[Type == 'AWS::[Service]::[Resource]']

rule [rule_name] when %[resource_type] !empty {
  %[resource_type].Properties {
    [Property] exists
    [Property] == [expected_value]
  }
}
```

### Guard Rule Examples

**Require S3 encryption:**
```ruby
let s3_buckets = Resources.*[Type == 'AWS::S3::Bucket']

rule s3_bucket_encryption_required when %s3_buckets !empty {
  %s3_buckets.Properties {
    BucketEncryption exists
    BucketEncryption.ServerSideEncryptionConfiguration[*] {
      ServerSideEncryptionByDefault exists
      ServerSideEncryptionByDefault.SSEAlgorithm in ["aws:kms", "aws:kms:dsse"]
    }
  }
}
```

**Require tags on EC2:**
```ruby
let ec2_instances = Resources.*[Type == 'AWS::EC2::Instance']

rule ec2_required_tags when %ec2_instances !empty {
  %ec2_instances.Properties {
    Tags exists
    Tags[*] {
      some Key == "CostCenter"
    }
    Tags[*] {
      some Key == "Environment"
    }
  }
}
```

---

## Authoring AWS Config Rules (Detective)

### Managed Rule (Preferred)

Use AWS managed Config rules when available. Search with:
```
aws___search_documentation: "AWS Config managed rule [resource] [property]"
```

Common managed rules:
- `S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED`
- `RDS_INSTANCE_PUBLIC_ACCESS_CHECK`
- `EC2_INSTANCE_NO_PUBLIC_IP`
- `REQUIRED_TAGS`
- `ENCRYPTED_VOLUMES`
- `IAM_USER_NO_POLICIES_CHECK`

### Custom Config Rule (CloudFormation Guard-based)

When no managed rule exists, create a custom rule using Guard:

```yaml
Type: AWS::Config::ConfigRule
Properties:
  ConfigRuleName: !Sub "kiro-power-${RuleName}"
  Description: !Sub "${Description}"
  Source:
    Owner: CUSTOM_POLICY
    SourceDetails:
      - EventSource: aws.config
        MessageType: ConfigurationItemChangeNotification
    CustomPolicyDetails:
      PolicyRuntime: guard-2.x.x
      PolicyText: |
        rule compliant_check {
          resourceType == "AWS::[Service]::[Resource]"
          configuration.[property] == [expected_value]
        }
  Scope:
    ComplianceResourceTypes:
      - "AWS::[Service]::[Resource]"
```

---

## CloudFormation Template Structure

Package all custom controls into a single CloudFormation template for StackSet deployment:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: !Sub "Governance Control: ${GovernanceIntent}"

Parameters:
  GovernanceControlName:
    Type: String
    Description: Name identifier for this governance control (must start with kiro-power-)

Resources:
  # Detective Control - Config Rule
  ComplianceConfigRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: !Sub "${GovernanceControlName}-detective"
      # ... config rule definition

  # Proactive Control - CloudFormation Guard Hook (if applicable)
  GuardHook:
    Type: AWS::CloudFormation::HookDefaultVersion
    Properties:
      # ... hook definition

Outputs:
  ConfigRuleArn:
    Value: !GetAtt ComplianceConfigRule.Arn
    Description: ARN of the deployed Config rule
```

**Note**: SCPs are NOT deployed via StackSets — they are created directly in the management account via Organizations API and attached to target OUs.

---

## Validation Before Deployment

Before presenting the deployment plan:

1. **Check policy quotas** — verify SCP count (<10) and RCP count (<5) on target OU. If at limit, cannot proceed with that policy type.
2. **Validate SCP syntax** — ensure valid JSON, no overly broad denies
3. **Validate Guard rules** — ensure they parse correctly
4. **Validate CloudFormation template** — use `aws___call_aws` with `cloudformation validate-template`
5. **Check for SCP conflicts** — ensure new SCP doesn't conflict with existing ones
6. **Verify SCP character limit** — individual SCP document max is 5,120 characters
