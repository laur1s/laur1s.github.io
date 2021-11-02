---
title: "Limit Access to AWS Regions With IAM and SCP"
date: 2021-11-02T15:48:39+02:00
---

By default AWS gives you access to all AWS regions. However, it's a very rare case that you might need to launch resources across all AWS regions in one account. In fact, I think is usually best to have one account per AWS region when possible. As some of the services are global (like IAM roles) by using one account per AWS region you can be sure that naming of IAM roles doesn't clash and you won't accidentally use an IAM role written for us-west-1 for your application in us-east-1 for example.

Another problem with having access to all AWS regions is that you might accidentally launch some resources in different AWS regions and then forget about these resources. If these other regions are not actively used the created resources might sit unnoticed and simply cost money.

## Limiting access to AWS regions

Now, let's look into ways for limiting access to AWS regions. There are two ways to restrict access in AWS: using IAM policies and AWS Organizations service control policies (SCPs). The differences between IAM and SCPs are described in detail in [this article by AWS](https://aws.amazon.com/premiumsupport/knowledge-center/iam-policy-service-control-policy/). In short, you'd use IAM to restrict or allow actions for IAM users groups or roles in a specific IAM account, and you can use SCPs to allow or deny access for members of AWS organization. Personally, I think using IAM policies and avoiding SCPs makes everything simpler but it won't work for everyone. I will discuss why I'm not a fan of SCPs in a future post as it would make this post too lengthy. Anyway, I will provide both SCP and IAM examples for restricting access to AWS regions next. It's possible to follow this blog post sequentially and create both a SCP and an IAM policy, or feel free to skip to the section that's relevant for you.

## Limiting access to AWS regions with a SCP

First, let's go to the AWS organizations console and enable Service Control Policies by clicking on Service Control Policies and enable:
{{< figure src="/images/iam/1.jpg" >}}
We can see that the default `FullAWSAccess` policy is available here but let's create a new policy that could limit access to AWS regions:
{{< figure src="/images/iam/2.jpg" >}}
In the next page let's enter a Policy name e.g. : `DenyAllOutsideEU`, description, and paste the following policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllOutsideEU",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*",
        "acm:*",
        "aws-marketplace-management:*",
        "aws-marketplace:*",
        "aws-portal:*",
        "budgets:*",
        "ce:*",
        "chime:*",
        "cloudfront:*",
        "config:*",
        "cur:*",
        "directconnect:*",
        "ec2:DescribeRegions",
        "ec2:DescribeTransitGateways",
        "ec2:DescribeVpnGateways",
        "fms:*",
        "globalaccelerator:*",
        "health:*",
        "iam:*",
        "importexport:*",
        "kms:*",
        "mobileanalytics:*",
        "networkmanager:*",
        "organizations:*",
        "pricing:*",
        "route53:*",
        "route53domains:*",
        "s3:GetAccountPublic*",
        "s3:ListAllMyBuckets",
        "s3:PutAccountPublic*",
        "shield:*",
        "sts:*",
        "support:*",
        "trustedadvisor:*",
        "waf-regional:*",
        "waf:*",
        "wafv2:*",
        "wellarchitected:*",
        "access-analyzer:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "eu-central-1",
            "eu-west-1"
          ]
        },
        "ArnNotLike": {
          "aws:PrincipalARN": [
            "arn:aws:iam::*:role/Role1AllowedToBypassThisSCP"
          ]
        }
      }
    }
  ]
}
```

It's quite a lengthy policy but essentially what it does is blocking access to all services if request comes from any other AWS regions than eu-west-1 and eu-central-1. There are also a long list of excepted services: they are usually global AWS services, or they might be some other exceptions (Cloudfront requires KMS in us-east-1 despite the region of CF distribution). If you don't plan to use these global services you can simply remove them from the policy for now. The example also adds a role that is allowed to bypass this SCP. For example, you could add the default `OrganizationAccountAccessRole` as an exception.

Next, is important to attach this policy by clicking on actions -> attach policy. I can attach the policy to the root of the organization, group of accounts, or a single AWS account. It's important to know that attaching this policy to the root account won't limit the permissions of root account because SPPs only work with member accounts!
{{< figure src="/images/iam/4.jpg" >}}
Now when I go to the RDS console in us-west-2 in my organization member account I receive a following error:
{{< figure src="/images/iam/3.jpg" >}}

## Limiting access to AWS regions with a IAM

Limiting access to AWS regions with IAM is pretty similar as with SCP. In this case, instead of working with the root account of organization I'll work with the AWS account where I want to disable access to certain AWS regions. To accomplish our goals I'll create a new IAM policy then attach this policy to `OrganizationAccountAccessRole`. Of course the policy can be attached to limit actions of any IAM role, user, or group in the account.

First, let's go to the IAM console -> Policies and create a new policy. Click on the Json tab to. We can copy paste the same policy from above, just deleting the ArnNotLike. We can also remove the `eu-central-1` from allowed regions. Now the policy only allows access from eu-west-1.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllOutsideEU",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*",
        "acm:*",
        "aws-marketplace-management:*",
        "aws-marketplace:*",
        "aws-portal:*",
        "budgets:*",
        "ce:*",
        "chime:*",
        "cloudfront:*",
        "config:*",
        "cur:*",
        "directconnect:*",
        "ec2:DescribeRegions",
        "ec2:DescribeTransitGateways",
        "ec2:DescribeVpnGateways",
        "fms:*",
        "globalaccelerator:*",
        "health:*",
        "iam:*",
        "importexport:*",
        "kms:*",
        "mobileanalytics:*",
        "networkmanager:*",
        "organizations:*",
        "pricing:*",
        "route53:*",
        "route53domains:*",
        "s3:GetAccountPublic*",
        "s3:ListAllMyBuckets",
        "s3:PutAccountPublic*",
        "shield:*",
        "sts:*",
        "support:*",
        "trustedadvisor:*",
        "waf-regional:*",
        "waf:*",
        "wafv2:*",
        "wellarchitected:*",
        "access-analyzer:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "eu-west-1"
          ]
        }
      }
    }
  ]
}
```

Next, let's attach this policy to `OrganizationAccountAccessRole` that we're currently assumed. In IAM Console, go to Roles, click on `OrganizationAccountAccessRole`. We can see that it currently has only one policy attached - `AdministratorAccess` (If you followed the SCP section and added the SCP policy, AdministratorAccess permissions are already constrained by SCP. Therefore, only eu-central-1 and eu-west-1 regions are allowed). Next, let's click on Attach Policies
{{< figure src="/images/iam/5.jpg" >}}

In the next window, find the AllowEUWest1Only policy and click attach policy. Now my role should be limited to eu-west-1 region only.

It's important to know that IAM policy changes takes some time to propagate. Therefore, immediately after attaching the policy I could be able to view the instances in eu-central-1 region. If that's the case, I wait some time or assume the role again.

I can see that my access to the ec2 console in eu-central-1 is blocked:
{{< figure src="/images/iam/6.jpg" >}}

## Conclusion

Now you should be familiar with 2 ways to block access to certain AWS regions: by using an IAM policy and a SCP policy. The advantage of SCP is that it limits access for all entities of the account. If using IAM the policy must be attached to all IAM entities that you wish to restrict. It's up to you on which approach you'd like to use. It's also important to know that AWS releases new services quite often and if these services are global you'd have to add them to the SCP or IAM policy. I created my SCP and IAM policies based on [this example](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_examples_general.html#example-scp-deny-region) on AWS documentation. You can always revert to this link and in case there are some changes to SCP it'll probably be updated.

Update: I mentioned to check AWS documentations for updates to the SCP when new services are released but actually I found out that AWS are not that great with updating the example documentation. `"access-analyzer:*"` was missing from the excepted services (It uses endpoint in us-east-1) so I added it to the both policies.
