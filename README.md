By completing this lab, you will be able to:

-   Understand what LocalStack is and why it is used in DevOps workflows
-   Install and run LocalStack locally using Docker
-   Configure AWS CLI to interact with a local AWS emulator
-   Work with core AWS services (S3, IAM, SQS) without using real AWS
-   Apply basic Infrastructure-as-Code thinking using CLI automation

------------------------------------------------------------------------

## 🏪 Scenario --- Le Café

You are part of the engineering team at **Le Café**, a coffee shop chain
migrating to AWS.

To avoid cloud costs and risks, the team uses **LocalStack**, a local
AWS emulator, to: - simulate AWS services - test infrastructure
locally - validate DevOps workflows safely

Your mission is to build and verify a complete local AWS environment.

------------------------------------------------------------------------

## 🧠 What is LocalStack?

LocalStack is a local AWS cloud emulator that runs inside a Docker
container.

It exposes AWS-compatible APIs at:

    http://localhost:4566

Instead of sending requests to real AWS, your AWS CLI and SDKs interact
with LocalStack.

### 💡 Key idea:

LocalStack = AWS simulator for development and testing (no cloud costs).

------------------------------------------------------------------------

## ⚙️ Prerequisites

Before running this project, ensure you have installed:

-   Docker Desktop
-   Python 3.8+
-   AWS CLI v2
-   LocalStack CLI
-   awscli-local (`awslocal`)