# CloudFormation Projects

A collection of AWS infrastructure defined declaratively with CloudFormation, instead of via the CLI or console.

**Portfolio Project | Infrastructure as Code Fundamentals**

---

## Project 1: S3 Bucket via CloudFormation

A CloudFormation template that provisions an S3 bucket declaratively.

### What It Does

This project defines an S3 bucket entirely in a YAML template, and lets AWS CloudFormation provision it from that single source of truth.

- Declares an S3 bucket as code (no manual console clicks)
- Deploys via `aws cloudformation create-stack`
- Verifies deployment status through the CLI
- Debugs real deployment failures using stack events
- Cleans up failed stacks and redeploys successfully

### CloudFormation & AWS Fundamentals Demonstrated

| Concept | How It's Used |
|---|---|
| YAML Templates | Defines the S3 bucket resource declaratively |
| CloudFormation Stacks | `create-stack`, `describe-stacks`, `delete-stack` |
| Stack Events | `describe-stack-events` to trace the real cause of failures |
| Resource Properties | `BucketName` configured under `Properties` |
| S3 Naming Rules | Bucket names must be globally unique and lowercase only |
| Git & GitHub | Repo created, cloned, and version-controlled from the start |
| AWS CLI | `aws configure`, `aws sts get-caller-identity` to verify authentication |

### How to Run

    aws cloudformation create-stack --stack-name my-s3-bucket-stack --template-body file://S3-Bucket.yaml

Then check deployment status:

    aws cloudformation describe-stacks --stack-name my-s3-bucket-stack --query "Stacks[0].StackStatus"

### Debugging Journey

Getting to `CREATE_COMPLETE` took four separate fixes:

1. **Duplicate YAML key** — `AWSTemplateFormatVersion` was declared twice (one with a typo), which YAML doesn't allow.
2. **VS Code Restricted Mode** — Source Control wouldn't activate until the workspace folder was explicitly trusted.
3. **Unsaved file** — The deploy failed with "At least one Resources member must be defined" because the file on disk didn't match what was visible in the editor. The fix was simply saving the file.
4. **Uppercase bucket name** — The stack reported success (returned a StackId) but actually rolled back. `describe-stack-events` revealed the real reason: S3 bucket names can't contain uppercase characters.

### Learning Outcomes

- How Infrastructure as Code differs from manually creating resources via CLI or console
- How to read CloudFormation stack statuses, and why a returned StackId doesn't guarantee success
- How to trace a deployment failure back to its root cause using `describe-stack-events`
- AWS-specific constraints (like S3's global uniqueness and lowercase-only naming) that don't show up until you actually deploy
- Why "it looks right in the editor" isn't the same as "it's saved to disk"

---

## Project 2: VPC with Subnets, Routing, and Internet Gateway

A single CloudFormation template that builds a custom VPC, splits it into six subnets across two Availability Zones, and wires up public internet connectivity.

### What It Does

- Provisions one VPC (`172.16.0.0/16`) with DNS support and hostnames enabled
- Defines six subnets across two Availability Zones: a public, app-private, and data-private subnet in each
- Creates an Internet Gateway and attaches it to the VPC
- Builds a public route table with a route to the internet (`0.0.0.0/0`)
- Associates both public subnets with that route table
- (In progress) Private subnet routing, to be added in a future update

### CloudFormation & AWS Fundamentals Demonstrated

| Concept | How It's Used |
|---|---|
| VPC & CIDR Blocks | `172.16.0.0/16` VPC split into six non-overlapping `/24` subnets |
| Dynamic AZ Selection | `!Select [0, !GetAZs '']` picks an AZ without hardcoding a region |
| Resource References | `!Ref` used throughout to link subnets, gateways, and route tables to the VPC |
| Internet Gateway | `AWS::EC2::InternetGateway` + `AWS::EC2::VPCGatewayAttachment` |
| Route Tables & Routes | `AWS::EC2::RouteTable`, `AWS::EC2::Route`, `AWS::EC2::SubnetRouteTableAssociation` |
| Resource Dependencies | `DependsOn` ensures the Internet Gateway is attached before the route is created |
| Stack Updates | `update-stack` used to add new resources without tearing down the existing VPC |
| Stack Drift | Deleting a resource outside CloudFormation (via the console) causes CloudFormation's records to fall out of sync with reality |

### How to Run

Deploy for the first time:

    aws cloudformation create-stack --stack-name vpc-stack --template-body file://vpc.yaml

Push new changes into the existing stack (without deleting and rebuilding it):

    aws cloudformation update-stack --stack-name vpc-stack --template-body file://vpc.yaml

Check status:

    aws cloudformation describe-stacks --stack-name vpc-stack

### Debugging Journey

This template involved more subtle bugs than the S3 project, structural and reference issues rather than single misspelled words:

1. **Unclosed quote** — A missing closing `'` on a `Description` line caused YAML to keep reading forward, so the reported error pointed to a completely different line than the actual mistake.
2. **Misspelled top-level key** — `AWSTemplateFormartVersion` instead of `AWSTemplateFormatVersion`.
3. **Property name casing** — `EnableDnsHostNames` instead of `EnableDnsHostnames`. Because the property wasn't recognized, CloudFormation rejected the entire template before it ever attempted to build the VPC resource, no resource-level event ever appeared in the stack's event log, only a vague top-level "Validation failed" message.
4. **Invalid intrinsic function syntax** — Wrote `!InternetGateway` instead of `!Ref InternetGateway` when referencing another resource by its logical ID.
5. **Wrong property name** — `InternetGateway` instead of `InternetGatewayId` on the `VPCGatewayAttachment` resource.
6. **Malformed resource type** — `AWS:EC2::SubnetRouteTableAssoction`, missing a colon (`AWS::EC2::...`) and misspelled (`Assoction` instead of `Association`).
7. **Broken YAML nesting** — `Type` and `Properties` were indented at the same level as their own resource name instead of one level deeper, making them siblings instead of children, which breaks the resource definition entirely.
8. **Stack drift** — Deleted the VPC directly from the AWS console instead of through CloudFormation. `describe-stacks` still reported `CREATE_COMPLETE` afterward, since CloudFormation only updates its own state when changes go through CloudFormation itself. Fixed by properly closing out the stack with `delete-stack` before redeploying.

### Learning Outcomes

- How CIDR notation works: the number after the slash determines how many of the 32 address bits are fixed vs. free, and how a VPC's larger range gets split into smaller, non-overlapping subnet ranges
- Why dynamically selecting an Availability Zone (`!GetAZs` + `!Select`) makes a template portable across regions, instead of hardcoding an AZ name
- The difference between a resource's **logical ID** (the key you choose in the template) and the **stack name** (chosen separately, at deploy time, and never written inside the template itself)
- That the same template can be deployed multiple times under different stack names to produce fully independent copies of the same infrastructure
- Why `update-stack` exists as its own command: it changes only what's different, without touching resources that don't need to change
- What "drift" means, and why deleting resources outside of CloudFormation causes CloudFormation's tracked state to disagree with what's actually in AWS
- How to read YAML indentation structurally, not just visually. Two keys at the same indentation level are siblings, not parent/child, even if that's not what was intended

---

## Built By

**Gustavo Lugo** | Cloud Engineering Student | Cloud Engineer Academy

## License

Open source — feel free to fork and modify!