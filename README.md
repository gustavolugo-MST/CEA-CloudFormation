# S3 Bucket via CloudFormation

A CloudFormation template that provisions an S3 bucket declaratively, instead of via the CLI or console.

**Portfolio Project | Built in a Single Session | Infrastructure as Code Fundamentals**

---

## What It Does

This project defines an S3 bucket entirely in a YAML template, and lets AWS CloudFormation provision it from that single source of truth.

- Declares an S3 bucket as code (no manual console clicks)
- Deploys via `aws cloudformation create-stack`
- Verifies deployment status through the CLI
- Debugs real deployment failures using stack events
- Cleans up failed stacks and redeploys successfully

---

## CloudFormation & AWS Fundamentals Demonstrated

| Concept | How It's Used |
|---|---|
| YAML Templates | Defines the S3 bucket resource declaratively |
| CloudFormation Stacks | `create-stack`, `describe-stacks`, `delete-stack` |
| Stack Events | `describe-stack-events` to trace the real cause of failures |
| Resource Properties | `BucketName` configured under `Properties` |
| S3 Naming Rules | Bucket names must be globally unique and lowercase only |
| Git & GitHub | Repo created, cloned, and version-controlled from the start |
| AWS CLI | `aws configure`, `aws sts get-caller-identity` to verify authentication |

---

## How to Run

    aws cloudformation create-stack --stack-name my-s3-bucket-stack --template-body file://S3-Bucket.yaml

Then check deployment status:

    aws cloudformation describe-stacks --stack-name my-s3-bucket-stack --query "Stacks[0].StackStatus"

---

## Debugging Journey

Getting to `CREATE_COMPLETE` took four separate fixes:

1. **Duplicate YAML key** — `AWSTemplateFormatVersion` was declared twice (one with a typo), which YAML doesn't allow.
2. **VS Code Restricted Mode** — Source Control wouldn't activate until the workspace folder was explicitly trusted.
3. **Unsaved file** — The deploy failed with "At least one Resources member must be defined" because the file on disk didn't match what was visible in the editor. The fix was simply saving the file.
4. **Uppercase bucket name** — The stack reported success (returned a StackId) but actually rolled back. `describe-stack-events` revealed the real reason: S3 bucket names can't contain uppercase characters.

---

## Learning Outcomes

By building this, I learned:

- How Infrastructure as Code differs from manually creating resources via CLI or console
- How to read CloudFormation stack statuses, and why a returned StackId doesn't guarantee success
- How to trace a deployment failure back to its root cause using `describe-stack-events`
- AWS-specific constraints (like S3's global uniqueness and lowercase-only naming) that don't show up until you actually deploy
- Why "it looks right in the editor" isn't the same as "it's saved to disk"

---

## Next Level Ideas

Want to expand this?

- Add parameters so the bucket name isn't hardcoded
- Add outputs to reference the bucket in other stacks
- Extend the template with IAM roles or bucket policies
- Convert to AWS CDK for a programmatic alternative to raw YAML
- Add a CI/CD pipeline to deploy on push

---

## Built By

**Gustavo Lugo** | Cloud Engineering Student | Cloud Engineer Academy

---

## License

Open source — feel free to fork and modify!
