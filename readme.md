# Privilege Escalation in AWS - Labs

## Prerequisites and useful tools

1. Install [awscli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) (AWS recommends python 3):
    1. Create an IAM account with Programmatic access in your account with the [AdministratorAccess](https://console.aws.amazon.com/iam/home#policies/arn:aws:iam::aws:policy/AdministratorAccess) managed policy, and download access keys
    2. Configure awscli on your machine by running `aws configure` with this new IAM Administrator Account
2. [Pacu](https://github.com/RhinoSecurityLabs/pacu) – _AWS exploitation framework_
3. [AWS-IAM-Permissions-Scanner](https://github.com/nemo-wq/AWS-IAM-Permissions-Scanner) – _Enumerate permissions for all users_
4. [Nimbostratus](https://github.com/andresriancho/nimbostratus) – _AWS Fingerprinting tool, dump permissions for given keys_

## Lab 1 - Introduction to IAM Accounts and Policies

1. Use your awscli with admin IAM account to create a new low privileged user
`aws iam create-user --user-name brucon1`
2. Create access keys, note the SecretAccessKey &amp; AccessKeyId
`aws iam create-access-key --user-name brucon1`
3. Setup awscli with a new profile, type in the newly generated AccessKey. Check [this link](https://docs.aws.amazon.com/general/latest/gr/rande.html) for region codes for your location.

`aws configure --profile brucon1`

You should be prompted for the following details:
```
AWS Access Key ID = _Generated above_
AWS Secret Access Key = _Generated above_
Default region name = ap-southeast-2
Default output format = None
```

Check if your awscli and configuration is working properly by the following steps using your admin IAM account:

  * Attach the SecurityAudit AWS Managed policy to the brucon1 user:

    `aws iam attach-user-policy --user-name brucon1 --policy-arn arn:aws:iam::aws:policy/SecurityAudit`

  * Check that the new profile using the brucon1 user profile (note the `--profile brucon1` parameter):

    `aws iam get-user --profile brucon1`
  
  * Verify Attached Policies as the default admin user:

    `aws iam list-attached-user-policies --user-name brucon1`

  * Detach Managed Policy:

    `aws iam detach-user-policy --user-name brucon1 --policy-arn arn:aws:iam::aws:policy/SecurityAudit`

  * Verify Attached Policies again as the default admin user:

    `aws iam list-attached-user-policies --user-name brucon1`



## Lab 2 – Privilege Escalation via policy versions

1. Check the appendix section and download the policy files.
1. Add a new Managed Policy to your AWS account called ```brucon-policy``` using your admin IAM account:

  `aws iam create-policy --policy-name brucon-policy --policy-document file://policy1.txt`

1. Create version 2 of brucon-policy:

  `aws iam create-policy-version --policy-arn arn:aws:iam::<your-account-number>:policy/brucon-policy --policy-document file://policy2.txt`

1. Create version 3 of the brucon-policy, and set as default:

  `aws iam create-policy-version --policy-arn arn:aws:iam::<your-account-number>:policy/brucon-policy --policy-document file://policy3.txt --set-as-default`

1. Attach the brucon-policy to brucon1 user which is sitting within your AWS account

  `aws iam attach-user-policy --user-name brucon1 --policy-arn arn:aws:iam::<your-account-number>:policy/brucon-policy`

The brucon1 user should now have limited permissions. The goal of Lab 1 is to identify the permissions using the below tools and achieve elevated privileges.

- https://github.com/nemo-wq/AWS-IAM-Permissions-Scanner
- http://andresriancho.github.io/nimbostratus/
- https://github.com/RhinoSecurityLabs/Security-Research/blob/master/tools/aws-pentest-tools/aws_escalate.py



## Lab 3 – Privilege Escalation via Accessing other User Accounts

1. Create new IAM user called brucon2 using your admin IAM account
```
aws iam create-user --user-name brucon2
aws iam create-access-key --user-name brucon2
aws configure --profile brucon2
```

1. Attach policy4
```
aws iam create-policy --policy-name brucon-policy2 --policy-document file://policy4.txt
aws iam attach-user-policy --user-name brucon2 --policy-arn arn:aws:iam::<your-account-number>:policy/brucon-policy2
```

1. Attempt to gain access to an admin account in your IAM store

## Lab 4 – Privilege Escalation via Group Membership

1. Create a new IAM group using your admin IAM account
  `aws iam create-group --group-name BruConGroup1`

1. Create a new user called brucon3, attach policy5, and add the user to the BruConGroup1 group
  `aws iam add-user-to-group --user-name brucon3 --group-name BruConGroup1`

1. Escalate privileges by attaching higher privileged policy to the BruConGroup1 group

## Lab 5 – Privilege Escalation via Misconfigured Role Policies

1. Create a new user called brucon4 using your admin IAM account
2. Attach policy6.txt to the brucon4 user
3. Create new role called BruConRole (easier using the management console)
4. Edit the role's trust policy to allow any user in your account to assume the BruConRole
    * Trust policy's principal element needs to be `"AWS": "arn:aws:iam::<your account number>:root"`
5. Update the policy associated with the BruConRole to gain administrative access

## Further Learning

1. Try completing the above exercises using [Pacu](https://github.com/RhinoSecurityLabs/pacu)
1. [CloudGoat](https://github.com/RhinoSecurityLabs/cloudgoat)
2. [flaws.cloud](http://flaws.cloud)
3. [flaws.cloud 2](http://flaws2.cloud)

## Appendix

### Lab Policies

https://github.com/nemo-wq/privilege_escalation/tree/master/labs



### Hints/Solutions

**Lab 2:** Check the previous policy versions of the brucon-policy customer managed policy. If any of the previous policy documents contains higher privileges, set that as default, otherwise create a new policy version, and set it as default. As this policy is already attached to the brucon1 user, the change in the policy document version will change the user&#39;s permissions. Try the following commands:

- `aws iam list-policy-versions`
- `aws iam get-policy-version`
- `aws iam set-default-policy-version`

**Lab 3:** The new user has a specific permission in policy4, which allows the user to create access keys for all other user accounts in the AWS account. Create an access key for the brucon1 user which should now have admin access, and only 1 set of IAM keys. Each user can have two access keys, therefore this should work as long as brucon1 user doesn't have two set of keys.

**Lab 4:** The group membership allows the brucon3 user to Attach policies to the Group. Privileges can be escalated by assigning an admin policy (like policy1.txt or the AWS AdministratorAccess policy to the group using `aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AdministratorAccess --group-name BruConGroup1`

**Lab 5:** You will need to use the iam:UpdateAssumeRolePolicy permission which your user has to modify the permissions associated with the BruConRole. Once done, assume the role and you should now have administrative access. Refer to [this](https://docs.aws.amazon.com/cli/latest/reference/iam/attach-role-policy.html) link for example usage.

