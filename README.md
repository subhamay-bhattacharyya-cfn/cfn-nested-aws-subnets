# CloudFormation Nested Stack Template : Subnets / Route Table / Nat Gateway

<!-- Row 1: Status - Most Important -->
[![Release](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)&nbsp;[![GitHub Repo](https://img.shields.io/badge/GitHub-Repository-blue?logo=github)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/github/languages/top/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazon&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-subnets/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/ffab152fbc481ed1bc4755596a44cd00/raw/cfn-nested-aws-subnets.json)](https://gist.github.com/subhamay-bhattacharyya/ffab152fbc481ed1bc4755596a44cd00)

## Overview

This repository contains a **nested stack template** (`cloudformation/template.yaml`) designed to be invoked from a parent/root CloudFormation stack. Upload the template to an S3 bucket and reference it with `AWS::CloudFormation::Stack` from the parent.

## Template Files

- **`cloudformation/template.yaml`** — Nested template for subnets, route tables and NAT gateways in a VPC
- **`cloudformation/parameters.json`** — Sample parameter values (`VpcName`, `SubnetQualifier`, `CiSuffix`) for CI deployment
- **`cloudformation/stack-config.json`** — Stack name, template file and parameter file used for deployment

## Subnet Template (cloudformation/template.yaml)

This nested stack template creates 1 to 6 subnets in a VPC, along with the route tables and NAT gateways they need. It is designed to be invoked from a parent stack. Subnet *N* is placed in the *N*th entry of `AvailabilityZones` and uses the *N*th entry of `SubnetCidrBlocks`.

### Routing by SubnetQualifier

| SubnetQualifier | Subnet Name tag | Default route | Route table |
| --------------- | --------------- | ------------- | ----------- |
| `public` | `{VpcName}-public-{AZ}` | `0.0.0.0/0` to the Internet Gateway (`InternetGatewayId`, or the CI Internet Gateway in CI mode) | One shared route table |
| `application` | `{VpcName}-application-pvt-{AZ}` | `0.0.0.0/0` to a NAT gateway in `NatPublicSubnetIds` | Shared, or one per AZ (see below) |
| `private` | `{VpcName}-private-pvt-{AZ}` | None | One shared route table |
| `database` | `{VpcName}-database-pvt-{AZ}` | None | One shared route table |

- Public subnets get `MapPublicIpOnLaunch: true`. All other subnets get `false`.
- When `CiSuffix` is set, `-{CiSuffix}` is appended to the Name tags.

### Application NAT Modes

`NatGatewayHighlyAvailable` applies only to `SubnetQualifier: application`.

- **`false`** (default): one Elastic IP and one NAT gateway, created in `NatPublicSubnetIds[0]`. All application subnets share one route table that routes `0.0.0.0/0` through it.
- **`true`**: one Elastic IP, NAT gateway and route table per AZ. NAT gateway *N* is created in `NatPublicSubnetIds[N-1]`, and application subnet *N* uses route table *N*. No shared route table is created.

### Deployment Modes

- **Existing VPC** (`CiSuffix` empty): `VpcId` is required. `InternetGatewayId` is required for `public` subnets. `NatPublicSubnetIds` is required for `application` subnets.
- **CI mode** (`CiSuffix` set): a VPC (`CiVpcCidrBlock`), an Internet Gateway and a public default route are created for the stack. `VpcId` must be left empty.

### Subnet Template Parameters

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `VpcId` | String | `""` | Existing VPC ID. Required when `CiSuffix` is empty. Must be empty when `CiSuffix` is set |
| `VpcName` | String | — | Value of the VPC Name tag, used as the prefix for Name tags (required) |
| `AvailabilityZones` | CommaDelimitedList | `us-east-1a,us-east-1b` | One AZ per subnet |
| `SubnetCidrBlocks` | CommaDelimitedList | `10.0.1.0/24,10.0.2.0/24` | One CIDR per subnet, in the same order as `AvailabilityZones` |
| `SubnetCount` | Number | `2` | Number of subnets to create (1–6) |
| `SubnetQualifier` | String | — | `public`, `private`, `application` or `database` (required) |
| `InternetGatewayId` | String | `""` | Internet Gateway ID. Required when `SubnetQualifier` is `public` and `CiSuffix` is empty |
| `CiSuffix` | String | `""` | Optional CI suffix (max 20 characters: lowercase letters, numbers, hyphens). Enables CI mode when set |
| `CiVpcCidrBlock` | String | `10.0.0.0/16` | CIDR of the VPC created in CI mode (`/16` to `/28`) |
| `NatGatewayHighlyAvailable` | String | `"false"` | `"true"` or `"false"`. See Application NAT Modes |
| `NatPublicSubnetIds` | CommaDelimitedList | `""` | Existing public subnet IDs that host the NAT gateways. Required when `SubnetQualifier` is `application`. In HA mode, list one per subnet in the same order as `AvailabilityZones` |

### Subnet Template Outputs

Outputs are not exported. Parent stacks read them with `!GetAtt <NestedStack>.Outputs.<Name>`.

| Output | Condition | Description |
| ------ | --------- | ----------- |
| `SubnetId1` to `SubnetId6` | `SubnetCount` ≥ N | ID of subnet N |
| `SubnetArn1` to `SubnetArn6` | `SubnetCount` ≥ N | ARN of subnet N (`arn:{partition}:ec2:{region}:{account}:subnet/{id}`) |
| `RouteTableId` | Shared route table exists | ID of the route table shared by all subnets. Not created for `application` with `NatGatewayHighlyAvailable: true` |

### Subnet Template Usage

Upload the template:

```bash
aws s3 cp cloudformation/template.yaml s3://your-cfn-bucket/templates/subnets.yaml
```

Reference it from a parent stack:

```yaml
SubnetsNestedStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/subnets.yaml
    Parameters:
      VpcId: !Ref VpcId
      VpcName: my-vpc
      AvailabilityZones: us-east-1a,us-east-1b
      SubnetCidrBlocks: 10.0.1.0/24,10.0.2.0/24
      SubnetCount: 2
      SubnetQualifier: application
      NatGatewayHighlyAvailable: "true"
      NatPublicSubnetIds: subnet-0aaa1111,subnet-0bbb2222
    Tags:
      - Key: Environment
        Value: !Ref Environment

Outputs:
  SubnetId1: !GetAtt SubnetsNestedStack.Outputs.SubnetId1
  SubnetArn1: !GetAtt SubnetsNestedStack.Outputs.SubnetArn1
```

Deploy the template directly to test it:

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name subnets-dev \
  --region us-east-1 \
  --parameter-overrides \
    VpcId=vpc-0123abcd \
    VpcName=my-vpc \
    SubnetQualifier=public \
    InternetGatewayId=igw-0123abcd
```

## Best Practices Implemented

- ✅ Private and database subnets have no default route to the internet or a NAT gateway
- ✅ Parameter validation with `AllowedPattern`, `AllowedValues` and rules (`VpcId` required without `CiSuffix`, `NatPublicSubnetIds` required for `application`)
- ✅ Optional highly available NAT gateway per AZ
- ✅ Subnet outputs are not exported, so parent stacks wire values explicitly

## License

MIT
