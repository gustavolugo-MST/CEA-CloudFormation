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
 
```
aws cloudformation create-stack --stack-name my-s3-bucket-stack --template-body file://S3-Bucket.yaml
```
 
Then check deployment status:
 
```
aws cloudformation describe-stacks --stack-name my-s3-bucket-stack --query "Stacks[0].StackStatus"
```
 
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
 
```
aws cloudformation create-stack --stack-name vpc-stack --template-body file://vpc.yaml
```
 
Push new changes into the existing stack (without deleting and rebuilding it):
 
```
aws cloudformation update-stack --stack-name vpc-stack --template-body file://vpc.yaml
```
 
Check status:
 
```
aws cloudformation describe-stacks --stack-name vpc-stack
```
 
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
 
## Project 3: Bastion Host, Private App Instances, and a Real Security Incident
 
Extends the same VPC from Project 2 with a bastion host pattern: a public-facing jump box, two private application instances reachable only through it, and security groups that model real network segmentation instead of one flat network.
 
### What It Does
 
- Deploys a `BastionHost` EC2 instance in a public subnet, reachable via SSH from one specific IP
- Deploys two private application instances (`App1`, `App2`) in separate Availability Zones, with no public IP at all
- Chains security groups so `App1` accepts SSH only from the bastion, and `App2` accepts ICMP (ping) only from `App1`
- Uses a CloudFormation Parameter for the bastion's allowed SSH source IP, so the template itself never contains a real IP address
- (Real incident) Detected, contained, and fully remediated a leaked SSH private key that was accidentally committed to a public GitHub repo
### CloudFormation & AWS Fundamentals Demonstrated
 
| Concept | How It's Used |
|---|---|
| Bastion Host Pattern | One public jump box (`BastionHost`) is the only entry point into private subnets |
| Security Group Chaining | `SourceSecurityGroupId` (not a CIDR) lets one security group reference another, so only traffic from a specific instance's SG is allowed |
| Least-Privilege Network Design | `App1`'s SG allows SSH only from `BastionSG`; `App2`'s SG allows ICMP only from `App1`'s SG, nothing else, from nowhere else |
| CloudFormation Parameters | `MyIpAddress` parameter removes a personal IP from the template entirely; supplied at deploy time via `--parameters` instead |
| EC2 Key Pairs | Key pairs are created outside CloudFormation (console/CLI) and referenced by name only, `!Ref` doesn't apply to resources that don't exist in the template |
| Resource Replacement | Some properties (like `KeyName`) can't be updated in place, changing them requires CloudFormation to terminate and relaunch the instance |
| Git History Rewriting | `git filter-repo` permanently removes a committed secret from every commit, not just the latest one |
 
### How to Run
 
```
aws cloudformation create-stack --stack-name vpc-stack --template-body file://vpc.yaml --parameters ParameterKey=MyIpAddress,ParameterValue=YOUR_CURRENT_IP/32
```
 
Update after any change, including after your IP rotates:
 
```
aws cloudformation update-stack --stack-name vpc-stack --template-body file://vpc.yaml --parameters ParameterKey=MyIpAddress,ParameterValue=YOUR_CURRENT_IP/32
```
 
Since `MyIpAddress` has no default value, CloudFormation refuses to deploy without it, on purpose.
 
### The Security Incident
 
While copying an EC2 key pair (`bastion.pem`) into the project folder for SSH access, it got swept up in a `git add .` and pushed to this public repo. Once caught:
 
1. **Assessed real exposure before panicking** — the bastion's security group only allowed SSH from one specific IP, and the instance had no IAM role attached, so the leaked key alone didn't hand out broad access.
2. **Rewrote git history** with `git filter-repo --path bastion.pem --invert-paths --force` to remove the file from every commit, not just delete it going forward, then force-pushed the cleaned history.
3. **Added a `.gitignore`** (`*.pem`) so this class of mistake can't repeat.
4. **Rotated the credential for real** — deleted the compromised key pair in AWS, created a new one, and ran a full `delete-stack` / `create-stack` to force every instance to relaunch with the new key, since the old public key was already baked into `authorized_keys` on the running instances and a name-only key rotation wouldn't have removed it.
5. **Hardened the template** so this couldn't happen the same way again — the personal IP that had also been committed became a `Parameter` instead of a hardcoded value.
### Debugging Journey
 
1. **Wrong resource `Type`** — `AppInstance1A`/`AppInstance2B` (actual EC2 instances) were briefly typed as `AWS::EC2::SecurityGroup`, a copy-paste artifact confirmed by independently spotting the instructor hit the identical bug.
2. **Invalid `!Ref` on a non-resource** — `KeyName: !Ref bastion` failed with "Unresolved resource dependencies," because the key pair was created outside CloudFormation and isn't a logical resource `!Ref` can resolve. Fixed with the plain string `bastion` instead.
3. **Free-tier instance type rejection** — `t2.micro` was rejected as ineligible for this account; `describe-instance-types --filters Name=free-tier-eligible,Values=true` surfaced `t3.micro` as a valid replacement.
4. **SSH timeout with no error at all** — `ssh` hung and eventually timed out with zero response, no host-key prompt, nothing. Root cause: a security group silently drops non-matching traffic instead of rejecting it, and a home ISP's dynamic IP had rotated since the security group was last deployed. Diagnosed by comparing `curl -4 ifconfig.me` against the CIDR in the template.
5. **Broken YAML nesting (again)** — two security group resources were indented one level too deep, nesting them as properties of another resource instead of siblings under `Resources:`.
### Learning Outcomes
 
- How a bastion host pattern actually restricts access at the security-group level, using `SourceSecurityGroupId` instead of open CIDR ranges
- Why leaked credentials need to be evaluated for real blast radius (security group scope, attached IAM roles) rather than reacting with blanket panic or blanket dismissal
- That deleting a secret from a repo's latest commit does nothing on its own, `git filter-repo` (or equivalent) is required to remove it from history entirely
- Why some CloudFormation properties force full resource replacement instead of updating in place, and how to reason about which ones do
- How to move a value that shouldn't be hardcoded (a personal IP) into a `Parameter`, the same principle behind not hardcoding secrets in application code
- That a security group silently dropping traffic (timeout) and actively rejecting it (immediate refusal) are different failure signatures worth telling apart when debugging
---
 
## Project 4: IAM Users, Groups, Roles, and Policies
 
A standalone CloudFormation template that models a small IAM setup: a user, a group, an EC2-assumable role, and a custom inline policy, covering both AWS-managed and self-authored permissions in the same file.
 
### What It Does
 
- Creates an IAM User (`GustavoCFN`) and IAM Group (`GustavosGroup`), and adds the user to the group via its own resource
- Creates an IAM Role (`MyIAMRole`) for EC2, with a trust policy allowing only EC2 instances to assume it
- Creates a custom inline IAM Policy granting `s3:GetObject`, attached directly to the role
- Attaches both AWS-managed policies (`AdministratorAccess`, `PowerUserAccess`) and a custom-authored policy side by side, to compare the two approaches directly
### CloudFormation & AWS Fundamentals Demonstrated
 
| Concept | How It's Used |
|---|---|
| IAM User / Group / Role | `AWS::IAM::User`, `AWS::IAM::Group`, `AWS::IAM::Role` |
| Group Membership | `AWS::IAM::UserToGroupAddition` attaches a user to a group as its own resource, not a property of either |
| Trust Policies vs. Permission Policies | `AssumeRolePolicyDocument` controls *who* can assume a role; `ManagedPolicyArns` / inline `Policies` control *what* it can do once assumed |
| Managed vs. Inline Policies | `ManagedPolicyArns` references existing, reusable AWS policies by ARN; `AWS::IAM::Policy` defines a custom, one-off policy directly in the template |
| IAM Policy Document Structure | `Version`, `Statement`, `Effect`, `Action`, `Resource`, `Principal` are fixed IAM policy-language fields, not user-chosen names |
| YAML Flow vs. Block Style | `['ec2.amazonaws.com']` and an equivalent dashed list produce identical data, two ways to write the same list |
 
### Debugging Journey
 
1. **Empty file deploy** — `Invalid length for parameter TemplateBody, value: 0`, the file had unsaved editor changes that hadn't been written to disk, the same class of bug from the very first S3 exercise.
2. **Broken command syntax** — wrote `--create-stack iam-stack` instead of `create-stack --stack-name iam-stack`; `create-stack` is a subcommand, not a flag.
3. **Broken nesting in the trust policy** — `Service` was a sibling of `Principal` instead of nested inside it; a follow-up fix then accidentally pulled `Action` in as a child of `Principal` too, instead of leaving it a sibling.
4. **Stray unmatched quote** — `sts:AssumeRole'` had a trailing apostrophe with no opening match, which would have been sent to AWS as a literal, invalid action name.
5. **A whole resource defined outside `Resources:`** — `MyIAMPolicy` was written at zero indentation, making it a sibling of `Resources:` itself instead of a resource inside it, the most severe of the nesting bugs.
6. **`PolicyDocument` written as a single malformed line** — needed `Version` and `Statement` nested inside it as their own block, not a flat string.
7. **Misplaced property** — `Roles` was nested inside an individual `Statement` entry instead of at the policy resource's top level, where it actually belongs.
8. **Typo** — `S3:GrtObject` instead of `S3:GetObject`, an invalid IAM action name.
### Learning Outcomes
 
- The difference between a resource's logical ID (freely chosen, never documented) and its `Type` value (fixed, AWS-defined, and what actually has a documentation page), and by extension, the difference between property *names* (always fixed) and property *values* (sometimes free-form, sometimes a fixed enum, sometimes a literal constant like `Version: '2012-10-17'`)
- Trust policies and permission policies answer two different questions entirely: *who can become this role* versus *what can this role do*
- Why IAM roles issue temporary, auto-rotating credentials instead of permanent ones, unlike a long-lived credential such as an IAM user's access key or an EC2 key pair
- How to find AWS's official documentation for a resource by searching its `Type` value, since the logical ID is never documented anywhere
- That YAML's bracket (flow-style) and dash (block-style) list syntaxes are functionally identical, just different formatting for the same underlying data
---
 
## Project 5: Application Load Balancer Across Two Public Web Servers
 
Extends the Project 2 VPC with an Application Load Balancer distributing HTTP traffic across two EC2 web server instances in separate Availability Zones, instead of exposing a single instance directly to the internet.
 
### What It Does
 
- Deploys an Application Load Balancer (`MyLoadBalancer`) spanning both public subnets
- Creates a Target Group and registers two EC2 instances (`WebServerInstance1A`, `WebServerInstance2B`) as static targets
- Creates a Listener forwarding port 80 traffic from the ALB to the Target Group
- Bootstraps both instances via `UserData` to install and start `httpd`, each serving a distinct message so traffic distribution is visible
### CloudFormation & AWS Fundamentals Demonstrated
 
| Concept | How It's Used |
|---|---|
| Application Load Balancer | `AWS::ElasticLoadBalancingV2::LoadBalancer` spans multiple public subnets for high availability |
| Target Groups | `AWS::ElasticLoadBalancingV2::TargetGroup` with static `Targets` (`Id: !Ref <instance>`) registers specific instances directly |
| Listeners | `AWS::ElasticLoadBalancingV2::Listener` connects the load balancer to the target group on a given port/protocol |
| Resource Reference Order | `MyTargetGroup` references EC2 instances defined later in the file; CloudFormation resolves dependencies from the whole template, not top-to-bottom file order |
| Security Group Scope | Sharing one security group between the ALB and the instances behind it means the instances stay directly reachable from the internet, not just through the load balancer |
 
### How to Run
 
```
aws cloudformation create-stack --stack-name ec2-stack --template-body file://ec2.yaml
```
 
### Learning Outcomes
 
- How an ALB, Target Group, and Listener function as three separate resources working together, rather than one monolithic "load balancer" resource
- That CloudFormation builds a full dependency graph from the template, so a resource can reference another resource defined further down in the same file without issue
- Why sharing a security group between a load balancer and the instances behind it undermines the point of having a load balancer in front at all, the same `SourceSecurityGroupId` chaining lesson from the bastion project (ALB security group open to the internet, instance security group open only to the ALB's security group) applies here too
- That load balancers bill continuously per hour plus usage, unlike IAM resources, worth factoring into how long a stack is left running
---
 
## Project 6: Auto Scaling Group with Launch Template, CloudWatch Alarm, and Scaling Policy
 
Builds on the same VPC and web server security group to add a Launch Template, an Auto Scaling Group spanning two public subnets, a CloudWatch alarm watching CPU utilization, and a scaling policy that reacts to it.
 
### What It Does
 
- Defines a Launch Template (`MyLaunchTemplate`) specifying the AMI, instance type, security group, and `httpd` bootstrap script for instances the ASG will launch
- Creates an Auto Scaling Group (`MyAutoScalingGroup`) with `MinSize: 1`, `MaxSize: 3`, `DesiredCapacity: 2`, spread across both public subnets
- Creates a CloudWatch Alarm (`HighCPUAlarm`) that watches average `CPUUtilization` for the group and trips above a 70% threshold
- Creates a Scaling Policy (`ScaleOutPolicy`) using `SimpleScaling` that the alarm triggers, adding one instance per trigger
### CloudFormation & AWS Fundamentals Demonstrated
 
| Concept | How It's Used |
|---|---|
| Launch Templates vs. Launch Configurations | AWS blocks new Launch Configuration creation entirely for accounts created on or after October 1, 2024, `AWS::EC2::LaunchTemplate` is the current replacement |
| Auto Scaling Groups | `AWS::AutoScaling::AutoScalingGroup` references a launch template by `LaunchTemplateId` and `Version` (`!GetAtt MyLaunchTemplate.LatestVersionNumber`) instead of a Launch Configuration name |
| CloudWatch Alarms | `AWS::CloudWatch::Alarm` watches a `Namespace`/`MetricName` pair (`AWS/EC2` / `CPUUtilization`) and triggers `AlarmActions` when a `Threshold` is crossed |
| One Resource Type, Multiple Strategies | `AWS::AutoScaling::ScalingPolicy` covers `SimpleScaling`, `StepScaling`, and `TargetTrackingScaling` in a single resource type; each strategy uses a different subset of properties, not all of them at once |
| Logical ID vs. Physical Resource Name | An Auto Scaling Group's actual AWS-assigned name isn't its logical ID unless `AutoScalingGroupName` is explicitly set; `describe-stack-resource --logical-resource-id` maps between the two |
| Opt-In Metrics | Per-instance `CPUUtilization` (used by the alarm) is always published; group-level metrics like `GroupInServiceInstances` require explicitly enabling `MetricsCollection` |
 
### How to Run
 
```
aws cloudformation create-stack --stack-name asg-stack --template-body file://asg.yaml
```
 
Find the ASG's real AWS-assigned name (not its logical ID) for CLI lookups:
 
```
aws cloudformation describe-stack-resource --stack-name asg-stack --logical-resource-id MyAutoScalingGroup --query "StackResourceDetail.PhysicalResourceId" --output text
```
 
### Debugging Journey
 
1. **Logical ID copied into `Type`** — `Type: AWS::AutoScaling::MyAutoScalingGroup` instead of the real, fixed AWS type `AWS::AutoScaling::AutoScalingGroup`. The logical ID (a name I invented) and the `Type` (a fixed AWS-defined string) got mixed up.
2. **Wrong property name** — `SecurityGroup` instead of the plural `SecurityGroups` on the original Launch Configuration.
3. **Malformed intrinsic function** — `Fn::Base64::` with a stray extra colon instead of `Fn::Base64:`.
4. **Two property-name typos in the alarm** — `CPUUtlization` instead of `CPUUtilization`, and `Thresdhold` instead of `Threshold`. The second one isn't a silent typo, CloudFormation rejects an unrecognized property outright.
5. **Broken YAML nesting** — `ScaleOutPolicy` was indented as a child of `HighCPUAlarm` instead of as its own sibling resource under `Resources:`, so the `!Ref ScaleOutPolicy` in the alarm's `AlarmActions` pointed at a resource that didn't actually exist as far as CloudFormation was concerned.
6. **Property casing** — `CoolDown` instead of the correct `Cooldown` on the scaling policy.
7. **Platform-level deprecation, not a template bug** — `create-stack` failed with "The Launch Configuration creation operation is not available in your account," because AWS blocks new Launch Configurations entirely for accounts created after October 1, 2024. Migrating to `AWS::EC2::LaunchTemplate` (different property names, like `SecurityGroupIds` instead of `SecurityGroups`, and a nested `LaunchTemplateData` block) was required regardless of how correct the original template was.
8. **Nesting bug again, mid-migration** — `MyLaunchTemplate`'s `Type` and `Properties` were indented at the same level as the resource name itself, making them siblings instead of children.
9. **Unsaved file before redeploy** — the same "editor shows it, disk doesn't have it" bug from the very first S3 exercise, `create-stack` was reading a stale, broken version of the file.
10. **Stack stuck in `ROLLBACK_COMPLETE`** — a failed `create-stack` leaves a stack that can't be updated in place; required `delete-stack` and a fresh `create-stack` after fixing the template, the same recovery pattern as the bastion key rotation.
### Learning Outcomes
 
- That a single CloudFormation resource type can bundle several genuinely different strategies together (`SimpleScaling` / `StepScaling` / `TargetTrackingScaling`), and the documentation lists every possible property across all of them, not a required checklist for any one of them
- That AWS deprecates and eventually blocks entire resource types at the platform level, independent of whether a template is written correctly, and CloudFormation error messages usually say so directly when that's the cause
- The difference between a resource's logical ID and its AWS-assigned physical name, and how to look up the real name with `describe-stack-resource` instead of guessing
- That CloudWatch has two separate tiers of metrics for an Auto Scaling Group, always-on per-instance metrics (what the alarm actually used) and opt-in group-level metrics (a console/`MetricsCollection` toggle, unrelated to whether scaling itself works)
- That compute-backed stacks (ALB, EC2, ASG) bill continuously and are worth tearing down once reviewed, unlike IAM-only stacks which cost nothing to leave running
---
 
## Project 7: S3 Static Website Hosting
 
A standalone S3 bucket configured to serve a static HTML page directly, with a bucket policy granting public read access, no EC2, no load balancer, just the bucket itself as the web server.
 
### What It Does
 
- Creates an S3 bucket (`MyS3Bucket`) with `WebsiteConfiguration` pointing to `index.html` as the index document
- Creates a separate `AWS::S3::BucketPolicy` resource granting public `s3:GetObject` on every object in the bucket
- Explicitly overrides specific `PublicAccessBlockConfiguration` settings so the public policy can actually attach and take effect
- Uploads `index.html` to the bucket as a separate step after the stack deploys, since CloudFormation provisions the bucket but doesn't move files into it
### CloudFormation & AWS Fundamentals Demonstrated
 
| Concept | How It's Used |
|---|---|
| Static Website Hosting | `WebsiteConfiguration` on `AWS::S3::Bucket` turns a bucket into an HTTP-servable static site, and its own distinct endpoint (`bucket.s3-website-region.amazonaws.com`) auto-resolves `IndexDocument` for bare directory paths |
| Bucket Policies | `AWS::S3::BucketPolicy` is its own resource, referencing a bucket by `!Ref`, rather than a property on the bucket resource itself |
| Public Access Block, Account-Level Default | As of April 28, 2023, all four `PublicAccessBlockConfiguration` settings default to blocking public access on any new bucket; a public website requires explicitly disabling the specific ones that apply |
| BlockPublicPolicy vs. RestrictPublicBuckets | Two different jobs: one blocks a public policy from being *attached* at all, the other restricts *access* even if a public policy exists. Disabling only one isn't enough on a post-2023 account |
| CloudFormation's Actual Scope | CloudFormation provisions the bucket and its configuration; it does not upload objects into it, `aws s3 cp` is a separate, necessary step |
| S3 Website Endpoint vs. Object Endpoint | The same public object is reachable at two different URL styles; only the website endpoint (not the plain `s3.region.amazonaws.com` object URL) auto-resolves an index document for bare paths |
 
### How to Run
 
```
aws cloudformation create-stack --stack-name s3-web-stack --template-body file://s3-static.yaml
```
 
Upload the site content separately, this doesn't happen automatically:
 
```
aws s3 cp index.html s3://<your-bucket-name>/index.html
```
 
### Debugging Journey
 
1. **Whole resource defined outside `Resources:`** — `MyS3Bucket` was written at zero indentation, making it a sibling of `Resources:` itself instead of a resource inside it, the same class of bug as `MyIAMPolicy` in the IAM project.
2. **Website inaccessible after a clean deploy (403 Forbidden)** — the bucket had `WebsiteConfiguration` and a working `BucketPolicy`, but only `RestrictPublicBuckets: false` was set under `PublicAccessBlockConfiguration`. `BlockPublicPolicy` was still defaulting to `true` (the account-level default since April 2023), which blocks a public bucket policy from attaching in the first place, not just from being read from. Adding `BlockPublicPolicy: false` alongside `RestrictPublicBuckets: false` fixed it.
3. **Instructor-provided code had the identical gap** — confirmed the same missing setting existed in the reference template too, a reminder that course material can be technically correct when written and still be outdated against a newer AWS account default, the same lesson as the Launch Configuration deprecation, just showing up on a different resource type.
4. **`describe-stack` vs. `describe-stacks`** — a command-name typo (missing the plural `s`), not a template issue; the CLI's own suggestion list pointed to the correct subcommand.
5. **Non-empty bucket blocking stack deletion** — `delete-stack` would have failed with `DELETE_FAILED` since the bucket still had `index.html` in it; CloudFormation won't delete a non-empty S3 bucket. Fixed by running `aws s3 rm s3://<bucket-name> --recursive` before deleting the stack.
### Learning Outcomes
 
- That S3's Block Public Access defaults changed at the platform level in April 2023, independent of anything in a given template, the exact same category of "written correctly once, broken by a later AWS default" issue as the Launch Configuration deprecation
- That `BlockPublicPolicy` and `RestrictPublicBuckets` solve two different problems (attaching a policy vs. honoring it) and disabling only one can look like partial success, a clean deploy followed by a 403, rather than an obvious failure
- That authority isn't evidence, instructor-provided code hit the identical gap, and the way to resolve a disagreement about whether code is correct is to test it, not to defer to who wrote it
- That CloudFormation's job ends at provisioning infrastructure, moving actual file content into that infrastructure is a separate, deliberate step
- That an S3 bucket must be emptied before its stack can be deleted, the same category of "CloudFormation won't act destructively without help" as `ROLLBACK_COMPLETE` stacks needing manual deletion before redeploying
---
 
## Project 8: RDS Database with Parameterized Credentials
 
A standalone MySQL RDS instance, built as a first introduction to managed databases in CloudFormation, with the master username and password moved into Parameters instead of hardcoded, following the same pattern established in the bastion project.
 
### What It Does
 
- Provisions a single MySQL RDS instance (`MyDB`) on `db.t4g.micro`
- Takes `DBUsername` and `DBPassword` as CloudFormation Parameters instead of hardcoding credentials into the template, with `NoEcho: true` on the password so it's masked from the console, `describe-stacks`, and stack events
- Supplies both values at deploy time via `--parameters` on the CLI, keeping the committed template itself free of any real credential material
### CloudFormation & AWS Fundamentals Demonstrated
 
| Concept | How It's Used |
|---|---|
| RDS Instances | `AWS::RDS::DBInstance` with `Engine`, `EngineVersion`, `DBInstanceClass`, and `AllocatedStorage` |
| Parameterized Credentials | `MasterUsername`/`MasterUserPassword` pull from `!Ref DBUsername`/`!Ref DBPassword` instead of literal strings, same pattern as the bastion project's `MyIpAddress` |
| `NoEcho` | Masks a parameter's value everywhere CloudFormation would otherwise display it (console, CLI output, events); it does not hide the value from shell history if typed directly into a command |
| Required Parameters, No Default | Neither parameter has a `Default`, so `create-stack` fails immediately and loudly if either is omitted, rather than deploying with a blank credential |
| RDS Networking Requirements | An RDS instance needs a VPC with at least two subnets across two Availability Zones to build an implicit DB Subnet Group; it does **not** require an explicitly declared security group, it falls back to the VPC's default security group automatically if none is specified |
| Account-Tier Service Limits | Free tier accounts cap `BackupRetentionPeriod` at 1 day; RDS enforces this at deploy time regardless of whether the template is otherwise valid |
 
### How to Run
 
```
aws cloudformation create-stack --stack-name rds-stack --template-body file://rds.yaml --parameters ParameterKey=DBUsername,ParameterValue=admin ParameterKey=DBPassword,ParameterValue=YourActualPassword
```
 
### Debugging Journey
 
1. **Misspelled top-level section** — `Resource:` instead of the required `Resources:`, which would have left CloudFormation seeing no resources at all in the template.
2. **Wrong property name** — `MasterPassword` instead of the actual property `MasterUserPassword`.
3. **Free tier backup retention limit** — `BackupRetentionPeriod: 7` failed with "exceeds the maximum available to free tier customers"; free tier caps this at 1 day, an account-level restriction rather than a template error.
4. **No default subnets in the account** — `create-stack` failed with "No default subnet detected in VPC," because every VPC in the account, including AWS's own automatic default VPC, had been deleted during earlier cleanup. `AWS::RDS::DBInstance` needs somewhere to place its network interface, and with zero VPCs left, it had nowhere to go.
5. **Recreating the default VPC, not just default subnets** — since the default VPC itself was gone (not just its subnets), `aws ec2 create-default-vpc` was the right tool, which rebuilds a fresh default VPC with a default subnet in every Availability Zone automatically, rather than manually recreating subnets one at a time with `create-default-subnet`.
6. **Mistaking a stale failure for a new one** — after fixing the VPC issue, `describe-stack-events` initially showed the exact same failure again, until noticing the Request ID was byte-for-byte identical to the earlier failed attempt. The stack hadn't actually been deleted and redeployed yet, it was the same old `ROLLBACK_COMPLETE` record being read again, not a fresh test of the fix.
7. **CLI syntax slip** — `git commit - m "..."` instead of `-m`, a space between the dash and the flag letter breaks it entirely.
### Learning Outcomes
 
- The precise reason RDS needs a VPC: to place its network interface across subnets in at least two Availability Zones, not because it strictly needs a security group explicitly declared, that part falls back to a sane default on its own
- That AWS enforces account-tier limits (like free tier backup retention caps) at the service level when the stack actually deploys, a category of failure that has nothing to do with whether the YAML is well-formed
- The difference between recreating a missing default *subnet* (`create-default-subnet`, when the VPC still exists) and recreating a missing default *VPC entirely* (`create-default-vpc`, when even that's gone), and why deleting "everything" during cleanup can remove infrastructure a later exercise silently depends on
- How to tell a genuinely new failure apart from a stale one, by checking whether the Request ID in the error actually changed between attempts, rather than assuming any redeploy happened just because a command was run
- What `NoEcho` actually protects (display surfaces) versus what it doesn't (shell history, terminal scrollback, process lists), and that authority (an instructor's own code) isn't a substitute for testing when two accounts might have different platform-level defaults
---
 
## Built By
 
**Gustavo Lugo** | Cloud Engineering Student | Cloud Engineer Academy
 
## License
 
Open source — feel free to fork and modify!