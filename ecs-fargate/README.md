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

Next steps — ALB + Auto-scaling

Architecture (internet-facing):

Internet
	|
	ALB (public)
	|
	Target Group (port 80)
	|
	ECS Service (Fargate tasks, private subnets)

How to implement (high level):
- Create `aws_lb` in public subnets and allow HTTP.
- Create an `aws_lb_target_group` for port 80 and configure health checks.
- Create an `aws_lb_listener` forwarding to the target group.
- Update `aws_ecs_service` to use private subnets and remove `assign_public_ip = true`.
- Add `load_balancer` block to the service so ECS registers tasks with the TG.
- Add `aws_appautoscaling_target` and `aws_appautoscaling_policy` to automatically change `desired_count`.

Short Terraform snippets (copy into `ecs-fargate/alb.tf` and `ecs-fargate/autoscaling.tf`):

ALB + TG + Listener:

```hcl
resource "aws_lb" "alb" { ... }
resource "aws_lb_target_group" "tg" { ... }
resource "aws_lb_listener" "http" { ... }
```

Autoscaling sample:

```hcl
resource "aws_appautoscaling_target" "ecs_target" { ... }
resource "aws_appautoscaling_policy" "cpu_scale_out" { ... }
```

If you want, I can scaffold `alb.tf` and `autoscaling.tf` now and adjust `main.tf` to stop assigning public IPs. Tell me whether you prefer I scaffold files or only update docs.

Get the ALB DNS name

After creating the ALB you can fetch its public DNS with:

```bash
aws elbv2 describe-load-balancers \
	--names hello-ecs-alb \
	--query 'LoadBalancers[0].DNSName' \
	--output text
```

