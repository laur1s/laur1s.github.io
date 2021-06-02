---
title: "Upgrading Terraform from 0.12 to 0.15"
date: 2021-06-02T12:28:46+03:00
---

If you follow Terraform best practices your Terraform infrastructure code should consist of multiple small modules that contain certain infrastructure code (in this case AWS).
However, the problem with smaller modules is that probably most of the infrastructure doesn't change that often. E.g.: once you create a VPC you don't need to update it very often unless there are
some changes with networking. On the other hand, other Terraform modules you use like ECS clusters, or ASG groups might require to be updated more often.

Terraform supports upgrade tools and features only for one major release upgrade at a time. That means that if you use `0.12` for your VPC module
you should first update to `0.13` then to `0.14` and finally to `0.15`. Therefore, I always try to update Terarform versions frequently.
However, for least frequently applied modules that's a bit more tricky. If you have 100s of modules that are used for your infrastructure is a bit of a chore to go throught each of them and run `terraform init` after every major update to terraform version.

I recently ran `terraform init` and received a following error:
```bash
$ terraform init
Initializing modules...
- alb

Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.
╷
│ Error: Invalid legacy provider address
│
│ This configuration or its associated state refers to the unqualified provider "aws".
│
│ You must complete the Terraform 0.13 upgrade process before upgrading to later versions.
╵
```

The issue happened because I already have `Terraform v0.15.2` on my system. I could've followed the update procedure and update from `0.12` to `0.15` incrementally but that looked like way to much of hassle. I found this command that replace the previously used Terraform AWS Provider name `aws` with the new name `hashicorp/aws`. Of course before running this command I downloaded a copy of my terraform state just to be safe.

```bash
$ terraform state replace-provider -- -/aws hashicorp/aws
Terraform will perform the following actions:

  ~ Updating provider:
    - registry.terraform.io/-/aws
    + registry.terraform.io/hashicorp/aws

Changing 2 resources:

  module.alb.aws_alb.this
  module.alb.aws_alb_listener.https

Do you want to make these changes?
Only 'yes' will be accepted to continue.

Enter a value: yes

Successfully replaced provider for 2 resources.
```

Now I can run `terraform init` without any errors:
```bash
$ terraform init
Initializing modules...
- alb

Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.
```

That's all for this short post. Hopefully, this will help for people that receive similar Terraform errors.

On the other note, I really like that Terraform now includes a lock file to pin the exact versions of Terraform providers. Since a lot of Infrastructure as Code (IaC) doesn't need to be updated
as frequently as new versions of Terraform itself perhaps something can be done to allow upgrading from Terraform version which is couple major versions behind of a newer release.
I know that because of sometimes painful update experience some companies choose to lock all their terraform code to specific version (like 0.12.2 but there are even companies that still use 0.11 in production). Hopefully, this can change as Terraform matures and becomes even better tool for IaC.
