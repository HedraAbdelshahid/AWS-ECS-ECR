# Hello ECS (Fargate) — Minimal Example

This repository contains a minimal static web app (served by `nginx` in a small Docker image) and a lightweight Terraform scaffold for deploying to AWS ECS Fargate.

Quick Docker / ECR workflow (adjust the account ID and region to your own):

1. Build the image (run from the repo root):

```bash
docker build -t hello-ecs:latest .
```

2. Authenticate Docker with ECR:

```bash
aws ecr get-login-password | docker login --username AWS --password-stdin 487037244797.dkr.ecr.us-east-1.amazonaws.com
```

3. Tag and push the image to ECR:

```bash
docker tag hello-ecs:latest 487037244797.dkr.ecr.us-east-1.amazonaws.com/hello-ecs:latest
docker push 487037244797.dkr.ecr.us-east-1.amazonaws.com/hello-ecs:latest
```

Notes:
- Replace `487037244797` and `us-east-1` with your AWS account ID and region.
- Do not store AWS credentials in this repo. Use environment variables or an AWS profile.
- The example image name is `hello-ecs:latest` and matches the Terraform example under `ecs-fargate`.

Terraform (see `ecs-fargate`):

```bash
cd ecs-fargate
terraform init
terraform plan
terraform apply
```

The Terraform configuration is intentionally minimal (provider + default VPC/subnets). To fully deploy a production-ready service you may need to add resources such as an ECR repository, IAM roles with least privilege, ALB/Target Group, and more.

See `ecs-fargate/README.md` for details.
