# ecs-fargate — Terraform scaffold

This folder contains a minimal Terraform scaffold for AWS provider configuration and lookups of the default VPC/subnets. It's intended as a starting point for deploying an ECS Fargate service.

Quick steps:

```bash
cd ecs-fargate
terraform init
terraform plan
terraform apply
```

Notes and gotchas:
- The current configuration is minimal: it includes provider settings and data lookups for the default VPC/subnets. It does not create an ECR repository, ALB, or advanced IAM roles.
- The example `main.tf` references the image `487037244797.dkr.ecr.us-east-1.amazonaws.com/hello-ecs:latest`. Push your Docker image to that repo (or update the image URI to your own account) before applying.
- Ensure AWS credentials are available via environment variables or an AWS profile.

Recommended next additions for a full deployment:
- `ecr.tf` to create an `aws_ecr_repository` resource and output the repo URI.
- `ecs.tf` to define an ECS cluster, task, and service (or expand the current files as needed).
- Proper IAM roles and policies with least privilege.
