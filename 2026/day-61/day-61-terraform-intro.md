# Day 61 – Terraform Introduction (Infrastructure as Code)

## Objective

Today I learned the fundamentals of **Terraform**, one of the most popular **Infrastructure as Code (IaC)** tools. Instead of manually creating AWS resources through the AWS Management Console, I learned how to define infrastructure using code, let Terraform create the resources, track their state, update them safely, and finally destroy them when no longer needed.

By the end of today's practice, I understood the complete Terraform workflow:

- Writing infrastructure in code
- Initializing a Terraform project
- Previewing infrastructure changes
- Creating AWS resources
- Understanding the Terraform state file
- Updating existing resources
- Destroying infrastructure safely

---

# Task 1 – Understanding Infrastructure as Code (IaC)

Before writing any Terraform code, I learned the concepts behind Infrastructure as Code.

## What is Infrastructure as Code?

Infrastructure as Code (IaC) is the process of managing infrastructure using code instead of manually creating resources through cloud consoles.

Instead of:

- Opening AWS Console
- Clicking buttons
- Filling forms
- Creating resources manually

I simply write configuration files describing the infrastructure I want, and Terraform automatically creates everything.

Example:

Instead of manually creating:

- EC2 Instance
- S3 Bucket
- VPC
- Security Groups

I define all of them inside a Terraform configuration file.

Terraform then communicates with AWS APIs and provisions everything automatically.

---

## Why IaC Matters in DevOps

Infrastructure is now treated exactly like application code.

Benefits include:

- Version controlled using Git
- Easy collaboration among team members
- Repeatable infrastructure creation
- Automated deployments
- Reduced human mistakes
- Easy disaster recovery
- Consistent environments

Without IaC:

Every developer may configure infrastructure differently.

With IaC:

Everyone creates exactly the same infrastructure every single time.

---

## Problems Solved by IaC

Manual AWS Console management has many drawbacks.

### Manual Infrastructure Problems

- Time consuming
- Easy to forget configuration
- Human errors
- Difficult to recreate environments
- No version history
- Hard to scale
- Poor collaboration

---

### IaC Solutions

Terraform solves these by making infrastructure:

- Repeatable
- Predictable
- Automated
- Version controlled
- Consistent
- Easy to review
- Easy to recreate

---

## Terraform vs Other Tools

### Terraform

Purpose:

Infrastructure Provisioning

Language:

HashiCorp Configuration Language (HCL)

Supports:

- AWS
- Azure
- Google Cloud
- Kubernetes
- GitHub
- Cloudflare
- Hundreds of providers

---

### AWS CloudFormation

AWS-only Infrastructure as Code service.

Uses:

- YAML
- JSON

Only works with AWS.

---

### Ansible

Primarily used for:

- Configuration Management
- Software Installation
- Server Configuration

Usually:

Terraform creates servers.

Ansible configures them.

---

### Pulumi

Infrastructure as Code using programming languages.

Languages include:

- Python
- Go
- JavaScript
- TypeScript
- C#

Instead of HCL.

---

## Declarative

Terraform is declarative.

That means I describe:

"What I want"

Instead of:

"How to do it"

Example:

```
I want one EC2 instance.
```

Terraform decides:

- Creation order
- Dependencies
- Required API calls

---

## Cloud Agnostic

Terraform is cloud agnostic.

The same Terraform workflow can manage:

- AWS
- Azure
- Google Cloud
- Kubernetes
- DigitalOcean

Only the provider changes.

---

# Task 2 – Creating My First Terraform Project

I created a file named:

```
main.tf
```

This file contained:

### Terraform Block

Specified:

- Required provider
- AWS provider source
- Version constraints

Example:

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

---

### Provider Block

Configured AWS region.

Example:

```hcl
provider "aws" {
  region = "ap-south-1"
}
```

---

### S3 Bucket Resource

Created an S3 bucket using Terraform.

Example:

```hcl
resource "aws_s3_bucket" "demo_bucket" {
  bucket = "my-unique-bucket-name"

  tags = {
    Name = "Terraform Demo"
  }
}
```

The bucket name had to be globally unique because S3 bucket names are shared across all AWS accounts.

---

# Terraform Lifecycle

## terraform init

Command:

```bash
terraform init
```

Purpose:

Initialized the Terraform working directory.

Terraform downloaded:

- AWS Provider Plugin
- Provider metadata
- Dependencies

Created:

```
.terraform/
```

and

```
.terraform.lock.hcl
```

---

## terraform plan

Command:

```bash
terraform plan
```

Purpose:

Previewed what Terraform planned to create.

Nothing was actually created.

Terraform compared:

Desired configuration

↓

Current infrastructure

↓

Displayed execution plan.

---

## terraform apply

Command:

```bash
terraform apply
```

Terraform displayed:

Execution Plan

↓

Asked for confirmation

↓

Created AWS resources.

After successful execution, I verified the S3 bucket inside AWS Console.

---

# Task 3 – Adding an EC2 Instance

Next, I expanded my infrastructure.

Added:

```hcl
resource "aws_instance" "terraform_ec2" {

  ami = "ami-xxxxxxxx"

  instance_type = "t2.micro"

  tags = {

    Name = "TerraWeek-Day1"

  }

}
```

Terraform Plan now showed:

```
Plan:

1 to add

0 to change

0 to destroy
```

Only the EC2 instance was created because the S3 bucket already existed.

---

## Why only EC2 was created?

Terraform maintains a state file.

It compared:

Current State

↓

Desired State

↓

Detected:

S3 already exists

EC2 missing

Therefore:

Only EC2 needed creation.

---

# Task 4 – Understanding Terraform State

Terraform automatically created:

```
terraform.tfstate
```

This file stores information about every managed resource.

Example:

- Resource ID
- ARN
- Attributes
- Tags
- Dependencies
- Current values

Terraform uses this file to determine future changes.

---

## terraform show

Command:

```bash
terraform show
```

Purpose:

Displays a human-readable version of the current Terraform state, including all managed resources and their attributes.

---

## terraform state list

Command:

```bash
terraform state list
```

Purpose:

Lists every resource currently managed by Terraform.

Example:

```
aws_instance.terraform_ec2

aws_s3_bucket.demo_bucket
```

---

## terraform state show

Commands:

```bash
terraform state show aws_s3_bucket.demo_bucket
```

and

```bash
terraform state show aws_instance.terraform_ec2
```

Purpose:

Displays detailed information about a specific resource stored in the state file, such as IDs, ARNs, tags, IP addresses, and other tracked attributes.

---

## Why should I never edit terraform.tfstate manually?

Because Terraform depends on this file to understand the infrastructure it manages.

Editing it manually can:

- Corrupt the state
- Cause resource duplication
- Lose track of resources
- Trigger unexpected updates or deletions

State changes should be made only through Terraform commands like:

- `terraform import`
- `terraform state mv`
- `terraform state rm`

---

## Why should terraform.tfstate not be committed to Git?

The state file may contain:

- Resource IDs
- ARNs
- Public IP addresses
- Private IP addresses
- Metadata
- Sensitive information (depending on the resources)

Committing it to Git can expose infrastructure details and cause conflicts if multiple people modify the same state. Production environments typically store the state in a remote backend (such as an S3 bucket with state locking) instead of version control.

---

# Task 5 – Modifying Infrastructure

I changed the EC2 Name tag.

From:

```
TerraWeek-Day1
```

To:

```
TerraWeek-Modified
```

Ran:

```bash
terraform plan
```

Terraform displayed:

```
~

update in-place
```

Meaning:

Only the tag changed.

The EC2 instance itself was **not recreated**.

---

## Terraform Symbols

| Symbol | Meaning |
|----------|----------|
| + | Create resource |
| ~ | Update existing resource |
| - | Destroy resource |
| -/+ | Destroy and recreate resource |

These symbols make it easy to understand what Terraform intends to do before applying changes.

---

# Applying Changes

Executed:

```bash
terraform apply
```

Terraform updated the EC2 tag.

Verified:

AWS EC2 Console

↓

Instance

↓

Name Tag

↓

Successfully changed.

---

# Destroying Infrastructure

Finally I cleaned up all created resources.

Command:

```bash
terraform destroy
```

Terraform displayed:

```
2 resources to destroy
```

After confirmation:

Terraform deleted:

- EC2 Instance
- S3 Bucket

I verified both resources were removed from the AWS Console.

---

# Important Terraform Commands Learned Today

Initialize project

```bash
terraform init
```

Preview execution plan

```bash
terraform plan
```

Create infrastructure

```bash
terraform apply
```

View current infrastructure

```bash
terraform show
```

List managed resources

```bash
terraform state list
```

View detailed state of one resource

```bash
terraform state show <resource_name>
```

Destroy infrastructure

```bash
terraform destroy
```

---

# Key Learnings

- Learned the concept of Infrastructure as Code (IaC).
- Understood why IaC is essential for DevOps automation.
- Compared Terraform with CloudFormation, Ansible, and Pulumi.
- Learned what "declarative" and "cloud-agnostic" mean in Terraform.
- Created my first `main.tf` configuration file.
- Configured the AWS provider and region.
- Provisioned an S3 bucket using Terraform.
- Provisioned an EC2 instance using Terraform.
- Learned the complete Terraform workflow: `init`, `plan`, `apply`, and `destroy`.
- Explored the purpose and structure of the `terraform.tfstate` file.
- Used `terraform show`, `terraform state list`, and `terraform state show` to inspect managed resources.
- Learned how Terraform tracks infrastructure using the state file.
- Modified an existing EC2 tag and observed an in-place update.
- Understood the meaning of Terraform plan symbols (`+`, `~`, `-`, and `-/+`).
- Safely destroyed all created infrastructure using `terraform destroy`.
- Gained a complete end-to-end understanding of how Terraform manages AWS infrastructure through code.
