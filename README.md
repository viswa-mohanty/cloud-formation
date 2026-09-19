# cloud-formation

Reusable AWS CloudFormation templates demonstrating infrastructure-as-code patterns with security scoped in by design, not left as an afterthought. Three templates, three different resource-policy shapes, one consistent principle: access is explicit and minimal, never a wildcard default.

## AWS_PrivateEndpoint.yaml

Creates an **Interface VPC Endpoint** (AWS PrivateLink) so traffic to a target AWS or partner service stays on the AWS network backbone instead of routing over the public internet or through a NAT gateway. This is the standard pattern for reaching services privately from inside a VPC without exposing that traffic externally.

### Design decision: least privilege by default

Interface endpoints support an optional policy that controls exactly what the allowed principals can do *through that endpoint* — separate from any IAM policy attached to those principals elsewhere. That's the whole value of an endpoint policy: without it, anyone who can reach the endpoint and who has broad IAM permissions can use it for broad access.

This template makes `AllowedActions` and `ResourceArns` **required parameters** rather than defaulting to `Action: '*'` / `Resource: '*'`. There's no wildcard fallback — whoever deploys this has to explicitly state which API actions and which resources the endpoint is allowed to reach. That's a deliberate choice: a template that silently defaults to "allow everything" is easy to deploy without thinking about scope, and endpoint policies are exactly the kind of control that gets left wide open when the path of least resistance is a wildcard.

### Parameters

| Parameter | Description |
|---|---|
| `VPCID` | The VPC to create the endpoint in |
| `SubnetIDs` | Comma-separated subnet IDs for the endpoint's network interfaces |
| `SecurityGroupId` | Security group controlling network-level access to the endpoint |
| `ServiceName` | The target PrivateLink service name (from AWS or the service provider) |
| `AWSPrincipals` | Comma-separated IAM users/roles/accounts allowed to use the endpoint |
| `AllowedActions` | **Required.** Comma-separated API actions those principals may call through this endpoint (e.g. `execute-api:Invoke`, or `s3:GetObject,s3:ListBucket`) — never `*` |
| `ResourceArns` | **Required.** Comma-separated resource ARNs the policy applies to — never `*` |

### Deploying

```bash
aws cloudformation deploy \
  --template-file AWS_PrivateEndpoint.yaml \
  --stack-name my-vpc-endpoint \
  --parameter-overrides \
    VPCID=vpc-0123456789abcdef0 \
    SubnetIDs=subnet-aaa,subnet-bbb \
    SecurityGroupId=sg-0123456789abcdef0 \
    ServiceName=com.amazonaws.vpce.us-east-1.vpce-svc-0123456789abcdef0 \
    AWSPrincipals=arn:aws:iam::111111111111:role/my-app-role \
    AllowedActions=execute-api:Invoke \
    ResourceArns=arn:aws:execute-api:us-east-1:111111111111:abc123/*
```

Substitute `AllowedActions` / `ResourceArns` with whatever the specific integration actually needs — that's the point of not hardcoding them.

## AWS_S3BucketPolicy.yaml

Creates an **S3 bucket** with encryption at rest (SSE-S3/AES256), versioning, all four Public Access Block settings enabled, and a bucket policy that combines two ideas:

1. **An explicit-allow statement** scoped to `AllowedActions`/`ResourceArns` — same pattern as the endpoint template, required parameters instead of a wildcard default.
2. **A deny-insecure-transport statement** that rejects any request not made over TLS (`aws:SecureTransport: false`).

That second statement uses `Principal: '*'` and `Action: 's3:*'` — which looks like it contradicts the "never wildcard" rule above, but it doesn't: this is a `Deny`, not an `Allow`. A wildcard `Deny` gated on a condition ("deny everyone, but only when they're not using TLS") is the standard AWS-recommended pattern for enforcing encryption in transit, and it's intentionally broad because the thing being denied is narrow and universally undesirable. The distinction that matters is `Allow` scope, not wildcards in the abstract.

### Deploying

```bash
aws cloudformation deploy \
  --template-file AWS_S3BucketPolicy.yaml \
  --stack-name my-secure-bucket \
  --parameter-overrides \
    BucketName=my-app-data-bucket \
    AWSPrincipals=arn:aws:iam::111111111111:role/my-app-role \
    AllowedActions=s3:GetObject,s3:PutObject \
    ResourceArns=arn:aws:s3:::my-app-data-bucket/*
```

## AWS_KMSKeyPolicy.yaml

Creates a **customer-managed KMS key** with key rotation enabled and a policy that separates three concerns: an `EnableIAMUserPermissions` statement delegating control to the account's IAM policies (the AWS-recommended default so the key never becomes unmanageable), a narrowly scoped `KeyAdministrators` statement for lifecycle actions (rotate, disable, delete, update policy), and a separate, narrower `KeyUsers` statement for actual encrypt/decrypt operations.

Every statement here uses `Resource: '*'` — and unlike the endpoint and bucket policies above, that's correct, not a shortcut. A KMS key policy is a resource-based policy already scoped to the one key it's attached to; there's no second resource dimension to narrow. The actual scoping happens on `Principal` (who) and `Action` (what they can do), which is why administrators and users are split into separate statements instead of one broad grant.

### Deploying

```bash
aws cloudformation deploy \
  --template-file AWS_KMSKeyPolicy.yaml \
  --stack-name my-app-data-key \
  --parameter-overrides \
    Alias=my-app-data-key \
    KeyAdministrators=arn:aws:iam::111111111111:role/security-admin \
    KeyUsers=arn:aws:iam::111111111111:role/my-app-role
```
