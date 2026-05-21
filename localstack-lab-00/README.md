# ☕ Lab 00 --- Discover AWS Locally with LocalStack

**Series:** Le Café --- AWS Hands-On Labs\
**Level:** Beginner\
**Duration:** \~60 minutes

------------------------------------------------------------------------

## 🎯 Learning Objectives
## 🚀 Step 1 --- Start LocalStack

``` bash
localstack start -d
```

Verify it is running:

``` bash
docker ps
```

Check services health:

``` bash
curl http://localhost:4566/_localstack/health
```

------------------------------------------------------------------------

## 🔑 Step 2 --- Configure AWS CLI

Create a local profile:

``` bash
aws configure --profile localstack
```

Use:

-   AWS Access Key ID: test\
-   AWS Secret Access Key: test\
-   Region: us-east-1

Activate:

``` bash
set AWS_PROFILE=localstack
```

------------------------------------------------------------------------

## 🪣 Step 3 --- S3 (Versioned Bucket)

``` bash
awslocal s3 mb s3://lecafe-menus
```

Enable versioning:

``` bash
awslocal s3api put-bucket-versioning --bucket lecafe-menus --versioning-configuration Status=Enabled
```

Upload versions:

``` bash
echo "Menu v1" > menu-paris.txt
awslocal s3 cp menu-paris.txt s3://lecafe-menus/menu-paris.txt

echo "Menu v2" > menu-paris.txt
awslocal s3 cp menu-paris.txt s3://lecafe-menus/menu-paris.txt
```

List versions:

``` bash
awslocal s3api list-object-versions --bucket lecafe-menus
```

------------------------------------------------------------------------

## 👤 Step 4 --- IAM

``` bash
awslocal iam create-user --user-name lecafe-app
awslocal iam create-group --group-name cafe-developers
awslocal iam add-user-to-group --user-name lecafe-app --group-name cafe-developers
awslocal iam attach-group-policy --group-name cafe-developers --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess
```

------------------------------------------------------------------------

## 📩 Step 5 --- SQS + DLQ

``` bash
awslocal sqs create-queue --queue-name lecafe-orders
awslocal sqs create-queue --queue-name lecafe-orders-dlq
```

Send message:

``` bash
awslocal sqs send-message --queue-url http://localhost:4566/000000000000/lecafe-orders --message-body "{"item":"Latte","size":"large","table":7}"
```

Receive:

``` bash
awslocal sqs receive-message --queue-url http://localhost:4566/000000000000/lecafe-orders
```

------------------------------------------------------------------------

## 🧹 Cleanup

``` bash
localstack stop
```
