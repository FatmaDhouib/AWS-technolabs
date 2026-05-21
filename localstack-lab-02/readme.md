\# Lab 02 — S3: Object Storage and Lifecycle Management



\## Series

Le Café — AWS Hands-On Labs with LocalStack



\## Level

Beginner → Intermediate



\## Estimated Duration

\~80 minutes



\---



\# Overview



This lab introduces Amazon S3 concepts through a practical LocalStack-based scenario. You will create and manage S3 buckets, configure versioning, apply bucket policies, host a static website, and automate object management using lifecycle rules.



The lab simulates a real-world environment where Le Café needs:



\- A public static website bucket

\- A private assets bucket with versioning enabled

\- A private logs bucket with automatic cleanup rules



\---



\# Learning Objectives



By the end of this lab, you will be able to:



\- Explain how S3 organises data and why it is not a traditional file system

\- Create and configure S3 buckets with versioning enabled

\- Upload, retrieve, and delete objects using the CLI and S3 API

\- Write bucket policies for access control

\- Configure lifecycle rules for automatic object expiration

\- Host a static website from an S3 bucket

\- Identify common S3 security misconfigurations



\---



\# Prerequisites



Before starting this lab, ensure that:



\- Lab 00 and Lab 01 are completed

\- LocalStack is installed and running

\- `awslocal` is configured correctly

\- AWS CLI is installed



\---



\# Scenario — Le Café’s Digital Presence



Le Café is expanding its digital infrastructure.



The marketing team requires a public static website to display seasonal menus.



The development team requires a private bucket for:



\- Application assets

\- Build artefacts

\- Configuration files



The operations team requires:



\- Versioning enabled on critical assets

\- Automatic deletion of old logs after 30 days



Your objective is to implement this architecture entirely within LocalStack.



\---



\# S3 Architecture



This lab uses three S3 buckets:



| Bucket Name | Purpose | Access |

|---|---|---|

| `lecafe-assets` | Application assets and configuration | Private |

| `lecafe-website` | Static website hosting | Public Read |

| `lecafe-logs` | Log storage with lifecycle management | Private |



\---



\# Understanding S3 Storage



S3 is not a traditional hierarchical file system.



S3 stores:



\- Buckets

\- Objects



Objects are identified by keys.



Example:



```text

menus/paris/summer-2026.pdf

```



This is not a real folder structure. The slashes are simply characters inside the object key.



Important implications:



\- Empty folders do not truly exist

\- Lifecycle rules work with prefixes

\- Object listings are flat key lists



A useful mental model is to think of S3 as a large key-value store.



\---



\# Part 1 — Create and Configure Buckets



\## Step 1 — Start LocalStack



```bash

localstack start -d

localstack status services



export AWS\_PROFILE=localstack

```



\---



\## Step 2 — Create the Buckets



```bash

\# Assets bucket

awslocal s3 mb s3://lecafe-assets



\# Website bucket

awslocal s3 mb s3://lecafe-website



\# Logs bucket

awslocal s3 mb s3://lecafe-logs

```



Verify bucket creation:



```bash

awslocal s3 ls

```



\---



\## Step 3 — Enable Versioning



Enable versioning on the assets bucket:



```bash

awslocal s3api put-bucket-versioning \\

&#x20; --bucket lecafe-assets \\

&#x20; --versioning-configuration Status=Enabled

```



Verify versioning:



```bash

awslocal s3api get-bucket-versioning \\

&#x20; --bucket lecafe-assets

```



Expected output:



```json

{

&#x20; "Status": "Enabled"

}

```



\---



\# Understanding `s3` vs `s3api`



| Command Family | Purpose |

|---|---|

| `awslocal s3` | High-level simplified operations |

| `awslocal s3api` | Direct low-level S3 API access |



Examples:



\- `cp`

\- `ls`

\- `mb`



belong to `s3`.



Advanced features like:



\- Versioning

\- Lifecycle rules

\- Bucket policies



require `s3api`.



\---



\# Part 2 — Work with Objects and Versioning



\## Step 4 — Upload Objects and Observe Versions



Create version 1:



```bash

echo "Le Café App Config — version 1" > config.txt



awslocal s3 cp config.txt \\

&#x20; s3://lecafe-assets/app/config.txt

```



Overwrite with version 2:



```bash

echo "Le Café App Config — version 2 (updated endpoint)" > config.txt



awslocal s3 cp config.txt \\

&#x20; s3://lecafe-assets/app/config.txt

```



List object versions:



```bash

awslocal s3api list-object-versions \\

&#x20; --bucket lecafe-assets

```



Retrieve an older version:



```bash

awslocal s3api get-object \\

&#x20; --bucket lecafe-assets \\

&#x20; --key app/config.txt \\

&#x20; --version-id VERSION\_ID \\

&#x20; recovered-config.txt

```



Display recovered content:



```bash

cat recovered-config.txt

```



\---



\## Step 5 — Understand Delete Markers



Delete the object:



```bash

awslocal s3 rm s3://lecafe-assets/app/config.txt

```



List objects:



```bash

awslocal s3 ls s3://lecafe-assets/app/

```



List versions and delete markers:



```bash

awslocal s3api list-object-versions \\

&#x20; --bucket lecafe-assets

```



Versioned buckets use delete markers instead of immediate deletion.



\---



\# Part 3 — Configure Bucket Policies



\## Step 6 — Restrict Access to the Assets Bucket



Create the bucket policy:



```bash

cat > /tmp/assets-bucket-policy.json << 'EOF'

{

&#x20; "Version": "2012-10-17",

&#x20; "Statement": \[

&#x20;   {

&#x20;     "Sid": "AllowAppRoleReadAccess",

&#x20;     "Effect": "Allow",

&#x20;     "Principal": {

&#x20;       "AWS": "arn:aws:iam::000000000000:role/lecafe-app-role"

&#x20;     },

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

&#x20;     "Sid": "DenyAllOtherPrincipals",

&#x20;     "Effect": "Deny",

&#x20;     "Principal": "\*",

&#x20;     "Action": "s3:\*",

&#x20;     "Condition": {

&#x20;       "ArnNotEquals": {

&#x20;         "aws:PrincipalArn": "arn:aws:iam::000000000000:role/lecafe-app-role"

&#x20;       }

&#x20;     }

&#x20;   }

&#x20; ]

}

EOF

```



Apply the policy:



```bash

awslocal s3api put-bucket-policy \\

&#x20; --bucket lecafe-assets \\

&#x20; --policy file:///tmp/assets-bucket-policy.json

```



Verify:



```bash

awslocal s3api get-bucket-policy \\

&#x20; --bucket lecafe-assets

```



\---



\## Step 7 — Configure Public Website Access



Create the public website policy:



```bash

cat > /tmp/website-bucket-policy.json << 'EOF'

{

&#x20; "Version": "2012-10-17",

&#x20; "Statement": \[

&#x20;   {

&#x20;     "Sid": "PublicReadForWebsite",

&#x20;     "Effect": "Allow",

&#x20;     "Principal": "\*",

&#x20;     "Action": "s3:GetObject",

&#x20;     "Resource": "arn:aws:s3:::lecafe-website/\*"

&#x20;   }

&#x20; ]

}

EOF

```



Apply the policy:



```bash

awslocal s3api put-bucket-policy \\

&#x20; --bucket lecafe-website \\

&#x20; --policy file:///tmp/website-bucket-policy.json

```



Enable static hosting:



```bash

awslocal s3api put-bucket-website \\

&#x20; --bucket lecafe-website \\

&#x20; --website-configuration '{

&#x20;   "IndexDocument": {"Suffix": "index.html"},

&#x20;   "ErrorDocument": {"Key": "error.html"}

&#x20; }'

```



\---



\# Part 4 — Host the Static Website



\## Step 8 — Create Website Files



Create `index.html`, `menu.html`, and `error.html`.



Upload the files:



```bash

awslocal s3 cp index.html s3://lecafe-website/index.html

awslocal s3 cp menu.html  s3://lecafe-website/menu.html

awslocal s3 cp error.html s3://lecafe-website/error.html

```



\---



\## Step 9 — Access the Website



Fetch the homepage:



```bash

curl http://localhost:4566/lecafe-website/index.html

```



Fetch the menu page:



```bash

curl http://localhost:4566/lecafe-website/menu.html

```



S3 serves the website content directly without requiring:



\- EC2

\- Nginx

\- Apache



\---



\# Part 5 — Lifecycle Rules



\## Step 10 — Why Lifecycle Rules Matter



Lifecycle rules automate storage management.



Common use cases:



\- Delete old logs

\- Transition data to cheaper storage classes

\- Control storage costs

\- Enforce retention policies



\---



\## Step 11 — Apply Lifecycle Configuration



Create lifecycle configuration:



```bash

cat > /tmp/logs-lifecycle.json << 'EOF'

{

&#x20; "Rules": \[

&#x20;   {

&#x20;     "ID": "DeleteOldLogs",

&#x20;     "Status": "Enabled",

&#x20;     "Filter": {

&#x20;       "Prefix": ""

&#x20;     },

&#x20;     "Expiration": {

&#x20;       "Days": 30

&#x20;     },

&#x20;     "NoncurrentVersionExpiration": {

&#x20;       "NoncurrentDays": 7

&#x20;     }

&#x20;   }

&#x20; ]

}

EOF

```



Apply the lifecycle rule:



```bash

awslocal s3api put-bucket-lifecycle-configuration \\

&#x20; --bucket lecafe-logs \\

&#x20; --lifecycle-configuration file:///tmp/logs-lifecycle.json

```



Verify configuration:



```bash

awslocal s3api get-bucket-lifecycle-configuration \\

&#x20; --bucket lecafe-logs

```



\---



\## Step 12 — Upload Sample Logs



Simulate log uploads:



```bash

for day in 01 02 03; do

&#x20; echo "\[2026-03-${day}] INFO: Order received — Latte x2, Croissant x1" \\

&#x20;   > app-log-${day}.txt



&#x20; awslocal s3 cp app-log-${day}.txt \\

&#x20;   s3://lecafe-logs/app-logs/2026-03-${day}.log

done

```



Verify uploads:



```bash

awslocal s3 ls s3://lecafe-logs/app-logs/

```



\---



\# Security Best Practices



\## Public Access Block



Always enable public access blocking unless a bucket must be public.



\---



\## Encryption at Rest



Recommended options:



\- SSE-S3

\- SSE-KMS



\---



\## Access Logging



Enable logging for audit and forensic investigations.



\---



\## Presigned URLs



Presigned URLs should:



\- Have short expiration times

\- Be treated like credentials

\- Never be shared carelessly



\---



\# Challenge Tasks



\## Challenge 1 — Glacier Transition Rule



Add a lifecycle rule that:



\- Transitions `audit-logs/` objects to GLACIER after 90 days

\- Deletes them after 365 days



\---



\## Challenge 2 — Presigned URL



Generate a presigned URL:



```bash

awslocal s3 presign \\

&#x20; s3://lecafe-assets/app/config.txt \\

&#x20; --expires-in 300

```



Use `curl` to access the object.



\---



\## Challenge 3 — Replication Simulation



Create a Bash script that:



1\. Lists objects from `lecafe-assets`

2\. Copies them into `lecafe-assets-backup`



Use:



```bash

awslocal s3api list-objects-v2

```



and



```bash

awslocal s3 cp

```



\---



\# Reflection Questions



1\. How would you balance version retention and storage cost in production?



2\. Why would CloudFront still be useful in front of a public S3 website bucket?



3\. Why are naming conventions important in flat S3 namespaces?



\---



\# Cleanup



\## Remove Versioned Objects



Delete all object versions:



```bash

awslocal s3api list-object-versions --bucket lecafe-assets \\

&#x20; --query 'Versions\[].{Key:Key,VersionId:VersionId}' \\

&#x20; --output text | while read key version; do

&#x20;   awslocal s3api delete-object \\

&#x20;     --bucket lecafe-assets \\

&#x20;     --key "$key" \\

&#x20;     --version-id "$version"

&#x20; done

```



Delete delete markers:



```bash

awslocal s3api list-object-versions --bucket lecafe-assets \\

&#x20; --query 'DeleteMarkers\[].{Key:Key,VersionId:VersionId}' \\

&#x20; --output text | while read key version; do

&#x20;   awslocal s3api delete-object \\

&#x20;     --bucket lecafe-assets \\

&#x20;     --key "$key" \\

&#x20;     --version-id "$version"

&#x20; done

```



\---



\## Remove Remaining Buckets



```bash

awslocal s3 rm s3://lecafe-website --recursive

awslocal s3 rm s3://lecafe-logs --recursive



awslocal s3 rb s3://lecafe-assets

awslocal s3 rb s3://lecafe-website

awslocal s3 rb s3://lecafe-logs

```



Stop LocalStack:



```bash

localstack stop

```



\---





