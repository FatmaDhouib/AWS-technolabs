\# Lab 01 — IAM: Identity and Access Management



Series: Le Café — AWS Hands-On Labs with LocalStack



Level: Beginner → Intermediate  

Duration: \~75 minutes



\## Prerequisites



\- Lab 00 completed

\- LocalStack installed and running

\- `awslocal` configured



\---



\# Learning Objectives



By the end of this lab, you will be able to:



\- Understand the difference between IAM users, groups, roles, and policies

\- Build a role-based access structure for a small organisation

\- Write custom IAM policies in JSON

\- Attach policies to groups using AWS best practices

\- Create and assume IAM roles

\- Understand temporary credentials and STS

\- Recognise common IAM security misconfigurations

\- Apply the principle of least privilege



\---



\# Scenario — Le Café Goes Cloudy



Le Café is expanding its cloud infrastructure.



The company now has three distinct access requirements:



\- Developers need access to upload and download assets from S3

\- Operations staff need visibility into infrastructure and logs

\- The ordering application itself needs programmatic AWS access



Your goal is to implement a secure IAM architecture using LocalStack.



\---



\# IAM Mental Model



IAM controls who can perform actions on AWS resources.



Every request to AWS is evaluated according to IAM policies.



IAM answers the question:



> Is this identity allowed to perform this action on this resource?



\## Core IAM Components



\### Users



Users represent human identities.



Examples:

\- alice

\- bob

\- charlie



Users typically have long-lived credentials.



\---



\### Groups



Groups are collections of users.



Policies are attached to groups instead of individual users.



Benefits:

\- Easier administration

\- Scalable permission management

\- Cleaner security model



\---



\### Policies



Policies are JSON documents defining permissions.



A policy contains:

\- Effect

\- Actions

\- Resources



Example structure:



```json

{

&#x20; "Version": "2012-10-17",

&#x20; "Statement": \[

&#x20;   {

&#x20;     "Effect": "Allow",

&#x20;     "Action": \["s3:GetObject"],

&#x20;     "Resource": "arn:aws:s3:::example-bucket/\*"

&#x20;   }

&#x20; ]

}

```



\---



\### Roles



Roles provide temporary credentials.



Roles are assumed by:

\- Applications

\- Services

\- Users



Roles are preferred over long-lived access keys.



\---



\# Architecture Overview



\## Access Model



\### Developers

\- Read/write access to S3 assets bucket

\- No database or billing access



\### Operations Team

\- Read-only EC2 visibility

\- Read CloudWatch logs

\- No modification permissions



\### Ordering Application

\- Read assets from S3

\- Send messages to SQS



\---



\# Part 1 — Create IAM Users



\## Step 1 — Start LocalStack



```bash

localstack start -d

localstack status services



export AWS\_PROFILE=localstack

```



\---



\## Step 2 — Create Users



```bash

\# Developers

awslocal iam create-user --user-name alice

awslocal iam create-user --user-name bob



\# Operations

awslocal iam create-user --user-name charlie

```



Verify users:



```bash

awslocal iam list-users

```



\---



\# Part 2 — Create Groups



\## Step 3 — Create Groups



```bash

awslocal iam create-group --group-name cafe-developers



awslocal iam create-group --group-name cafe-operations

```



\---



\## Step 4 — Add Users to Groups



```bash

\# Developers group

awslocal iam add-user-to-group \\

&#x20; --user-name alice \\

&#x20; --group-name cafe-developers



awslocal iam add-user-to-group \\

&#x20; --user-name bob \\

&#x20; --group-name cafe-developers



\# Operations group

awslocal iam add-user-to-group \\

&#x20; --user-name charlie \\

&#x20; --group-name cafe-operations

```



Verify memberships:



```bash

awslocal iam get-group --group-name cafe-developers



awslocal iam get-group --group-name cafe-operations

```



\---



\# Part 3 — Create Custom Policies



\## Step 5 — IAM Policy Structure



Basic IAM policy format:



```json

{

&#x20; "Version": "2012-10-17",

&#x20; "Statement": \[

&#x20;   {

&#x20;     "Sid": "StatementLabel",

&#x20;     "Effect": "Allow",

&#x20;     "Action": \["service:Action"],

&#x20;     "Resource": "resource-arn"

&#x20;   }

&#x20; ]

}

```



\## Important Fields



| Field | Description |

|---|---|

| Version | Policy language version |

| Statement | List of permission blocks |

| Sid | Optional identifier |

| Effect | Allow or Deny |

| Action | AWS API operations |

| Resource | Target AWS resource ARN |



\---



\## Step 6 — Developer S3 Policy



Create the policy file:



```bash

cat > /tmp/developer-s3-policy.json << 'EOF'

{

&#x20; "Version": "2012-10-17",

&#x20; "Statement": \[

&#x20;   {

&#x20;     "Sid": "AllowS3BucketListing",

&#x20;     "Effect": "Allow",

&#x20;     "Action": \[

&#x20;       "s3:ListBucket"

&#x20;     ],

&#x20;     "Resource": "arn:aws:s3:::lecafe-assets"

&#x20;   },

&#x20;   {

&#x20;     "Sid": "AllowS3ObjectOperations",

&#x20;     "Effect": "Allow",

&#x20;     "Action": \[

&#x20;       "s3:GetObject",

&#x20;       "s3:PutObject",

&#x20;       "s3:DeleteObject"

&#x20;     ],

&#x20;     "Resource": "arn:aws:s3:::lecafe-assets/\*"

&#x20;   }

&#x20; ]

}

EOF

```



Create the policy:



```bash

awslocal iam create-policy \\

&#x20; --policy-name LeCafe-Developer-S3 \\

&#x20; --policy-document file:///tmp/developer-s3-policy.json

```



\---



\## Step 7 — Operations Read-Only Policy



Create the policy file:



```bash

cat > /tmp/operations-policy.json << 'EOF'

{

&#x20; "Version": "2012-10-17",

&#x20; "Statement": \[

&#x20;   {

&#x20;     "Sid": "AllowEC2ReadOnly",

&#x20;     "Effect": "Allow",

&#x20;     "Action": \[

&#x20;       "ec2:DescribeInstances",

&#x20;       "ec2:DescribeInstanceStatus",

&#x20;       "ec2:DescribeRegions",

&#x20;       "ec2:DescribeSecurityGroups",

&#x20;       "ec2:DescribeVpcs"

&#x20;     ],

&#x20;     "Resource": "\*"

&#x20;   },

&#x20;   {

&#x20;     "Sid": "AllowCloudWatchReadOnly",

&#x20;     "Effect": "Allow",

&#x20;     "Action": \[

&#x20;       "logs:DescribeLogGroups",

&#x20;       "logs:DescribeLogStreams",

&#x20;       "logs:GetLogEvents",

&#x20;       "logs:FilterLogEvents"

&#x20;     ],

&#x20;     "Resource": "\*"

&#x20;   }

&#x20; ]

}

EOF

```



Create the policy:



```bash

awslocal iam create-policy \\

&#x20; --policy-name LeCafe-Operations-ReadOnly \\

&#x20; --policy-document file:///tmp/operations-policy.json

```



\---



\## Step 8 — Attach Policies to Groups



Attach developer policy:



```bash

awslocal iam attach-group-policy \\

&#x20; --group-name cafe-developers \\

&#x20; --policy-arn arn:aws:iam::000000000000:policy/LeCafe-Developer-S3

```



Attach operations policy:



```bash

awslocal iam attach-group-policy \\

&#x20; --group-name cafe-operations \\

&#x20; --policy-arn arn:aws:iam::000000000000:policy/LeCafe-Operations-ReadOnly

```



Verify attachments:



```bash

awslocal iam list-attached-group-policies \\

&#x20; --group-name cafe-developers



awslocal iam list-attached-group-policies \\

&#x20; --group-name cafe-operations

```



\---



\# Part 4 — Create a Service Role



\## Step 9 — Trust Policies



Roles contain:

\- Permission policies

\- Trust policies



Trust policies define who can assume the role.



\---



\## Step 10 — Create the Trust Policy



```bash

cat > /tmp/trust-policy.json << 'EOF'

{

&#x20; "Version": "2012-10-17",

&#x20; "Statement": \[

&#x20;   {

&#x20;     "Sid": "AllowEC2ToAssumeRole",

&#x20;     "Effect": "Allow",

&#x20;     "Principal": {

&#x20;       "Service": "ec2.amazonaws.com"

&#x20;     },

&#x20;     "Action": "sts:AssumeRole"

&#x20;   }

&#x20; ]

}

EOF

```



\---



\## Step 11 — Create the IAM Role



```bash

awslocal iam create-role \\

&#x20; --role-name lecafe-app-role \\

&#x20; --assume-role-policy-document file:///tmp/trust-policy.json \\

&#x20; --description "Role assumed by the Le Cafe ordering application"

```



\---



\## Step 12 — Attach Application Permissions



Create inline role policy:



```bash

cat > /tmp/app-role-policy.json << 'EOF'

{

&#x20; "Version": "2012-10-17",

&#x20; "Statement": \[

&#x20;   {

&#x20;     "Sid": "AllowS3AssetRead",

&#x20;     "Effect": "Allow",

&#x20;     "Action": \[

&#x20;       "s3:GetObject",

&#x20;       "s3:ListBucket"

&#x20;     ],

&#x20;     "Resource": \[

&#x20;       "arn:aws:s3:::lecafe-assets",

&#x20;       "arn:aws:s3:::lecafe-assets/\*"

&#x20;     ]

&#x20;   },

&#x20;   {

&#x20;     "Sid": "AllowSQSOrderWrite",

&#x20;     "Effect": "Allow",

&#x20;     "Action": \[

&#x20;       "sqs:SendMessage",

&#x20;       "sqs:GetQueueUrl",

&#x20;       "sqs:GetQueueAttributes"

&#x20;     ],

&#x20;     "Resource": "arn:aws:sqs:us-east-1:000000000000:lecafe-orders"

&#x20;   }

&#x20; ]

}

EOF

```



Attach the inline policy:



```bash

awslocal iam put-role-policy \\

&#x20; --role-name lecafe-app-role \\

&#x20; --policy-name LeCafe-App-Permissions \\

&#x20; --policy-document file:///tmp/app-role-policy.json

```



\---



\## Step 13 — Assume the Role



```bash

awslocal sts assume-role \\

&#x20; --role-arn arn:aws:iam::000000000000:role/lecafe-app-role \\

&#x20; --role-session-name ordering-app-session

```



This command returns temporary credentials:

\- AccessKeyId

\- SecretAccessKey

\- SessionToken

\- Expiration



\---



\# Part 5 — Verify Permissions



\## Step 14 — Create Supporting Resources



Create the S3 bucket:



```bash

awslocal s3 mb s3://lecafe-assets

```



Create the SQS queue:



```bash

awslocal sqs create-queue \\

&#x20; --queue-name lecafe-orders

```



\---



\## Step 15 — Inspect IAM Resources



View role configuration:



```bash

awslocal iam get-role \\

&#x20; --role-name lecafe-app-role



awslocal iam get-role-policy \\

&#x20; --role-name lecafe-app-role \\

&#x20; --policy-name LeCafe-App-Permissions

```



Inspect user permissions:



```bash

awslocal iam list-groups-for-user \\

&#x20; --user-name alice



awslocal iam list-attached-group-policies \\

&#x20; --group-name cafe-developers

```



Inspect policy details:



```bash

awslocal iam get-policy \\

&#x20; --policy-arn arn:aws:iam::000000000000:policy/LeCafe-Developer-S3



awslocal iam get-policy-version \\

&#x20; --policy-arn arn:aws:iam::000000000000:policy/LeCafe-Developer-S3 \\

&#x20; --version-id v1

```



\---



\# Security Notes



\## Principle of Least Privilege



Always grant only the permissions required.



Avoid:

```json

"Resource": "\*"

```



unless technically necessary.



\---



\## Explicit Deny Overrides Allow



AWS policy evaluation rules:

\- Explicit Deny always wins

\- Implicit Deny is the default

\- Allows must be explicitly granted



\---



\## Prefer Roles Over Access Keys



Roles provide:

\- Temporary credentials

\- Automatic rotation

\- Reduced credential leakage risk



Avoid hardcoding long-lived access keys in applications.



\---



\# Key Takeaways



In this lab you learned how to:



\- Create IAM users and groups

\- Design role-based access control

\- Write custom IAM policies

\- Attach policies to groups

\- Create IAM roles with trust policies

\- Use STS AssumeRole

\- Apply least privilege principles

\- Build secure cloud access patterns



\---



\# Cleanup (Optional)



Delete users:



```bash

awslocal iam delete-user --user-name alice

awslocal iam delete-user --user-name bob

awslocal iam delete-user --user-name charlie

```



Delete groups:



```bash

awslocal iam delete-group --group-name cafe-developers

awslocal iam delete-group --group-name cafe-operations

```



Delete policies:



```bash

awslocal iam delete-policy \\

&#x20; --policy-arn arn:aws:iam::000000000000:policy/LeCafe-Developer-S3



awslocal iam delete-policy \\

&#x20; --policy-arn arn:aws:iam::000000000000:policy/LeCafe-Operations-ReadOnly

```



Delete role:



```bash

awslocal iam delete-role \\

&#x20; --role-name lecafe-app-role

```

