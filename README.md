# cloud-formation

Reusable AWS CloudFormation templates demonstrating infrastructure-as-code patterns with security scoped in by design, not left as an afterthought.

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
