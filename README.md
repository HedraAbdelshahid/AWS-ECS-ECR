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

Next steps — ALB + Auto-scaling

Architecture (internet-facing):

Internet
	 |
	 ALB (public)
	 |
	 Target Group (port 80)
	 |
	 ECS Service (Fargate tasks, private subnets)

What to change in Terraform
- Create an Application Load Balancer (`aws_lb`) in public subnets and allow HTTP (port 80).
- Create an `aws_lb_target_group` for port 80 with a suitable health check.
- Create an `aws_lb_listener` that forwards to the target group.
- Update the `aws_ecs_service` `network_configuration` to use private subnets and remove `assign_public_ip = true`.
- Attach the ECS tasks to the target group by adding `load_balancer` block or via `service_registries`, depending on your setup.

Get the ALB DNS name

After creating the ALB you can fetch its public DNS with:

```bash
aws elbv2 describe-load-balancers \
	--names hello-ecs-alb \
	--query 'LoadBalancers[0].DNSName' \
	--output text
```

Autoscaling
- Add `aws_appautoscaling_target` and `aws_appautoscaling_policy` to scale the ECS service (task count) based on CPU or custom metrics.

Production-ready notes (what this repo now implements)

- `main.tf` references `aws_vpc.this` and separates public/private subnets for correct network placement.
- The ALB (`aws_lb`) is deployed into public subnets (internet-facing) and forwards to a target group on port 80.
- ECS Fargate tasks run in private subnets without public IPs; tasks are registered with the ALB target group so they receive traffic.
- A NAT gateway provides outbound internet access from private subnets so tasks can pull container images from ECR or other registries.
- Autoscaling is configured via `aws_appautoscaling_target` and `aws_appautoscaling_policy` to adjust `desired_count` automatically.

Before applying in a new account, verify that private subnets have a route to a NAT gateway in a public subnet (or add an appropriate NAT resource).


Minimal Terraform snippets (add to `ecs-fargate/*.tf`):

ALB + TG + Listener example:

```hcl
resource "aws_lb" "alb" {
	name               = "hello-alb"
	internal           = false
	load_balancer_type = "application"
	subnets            = data.aws_subnets.default.ids
}

resource "aws_lb_target_group" "tg" {
	name     = "hello-tg"
	port     = 80
	protocol = "HTTP"
	vpc_id   = data.aws_vpc.default.id
	health_check {
		path                = "/"
		interval            = 30
		healthy_threshold   = 2
		unhealthy_threshold = 2
	}
}

resource "aws_lb_listener" "http" {
	load_balancer_arn = aws_lb.alb.arn
	port              = "80"
	protocol          = "HTTP"
	default_action {
		type             = "forward"
		target_group_arn = aws_lb_target_group.tg.arn
	}
}
```

ECS service changes (use private subnets):

```hcl
network_configuration {
	subnets          = data.aws_subnets.default.ids # replace with private subnet ids
	security_groups  = [aws_security_group.ecs.id]
	assign_public_ip = false
}

load_balancer {
	target_group_arn = aws_lb_target_group.tg.arn
	container_name   = "hello"
	container_port   = 80
}
```

App autoscaling example:

```hcl
resource "aws_appautoscaling_target" "ecs_target" {
	max_capacity       = 4
	min_capacity       = 1
	resource_id        = "service/${aws_ecs_cluster.this.name}/${aws_ecs_service.this.name}"
	scalable_dimension = "ecs:service:DesiredCount"
	service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu_scale_out" {
	name               = "cpu-scale-out"
	policy_type        = "TargetTrackingScaling"
	resource_id        = aws_appautoscaling_target.ecs_target.resource_id
	scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
	service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

	target_tracking_scaling_policy_configuration {
		predefined_metric_specification {
			predefined_metric_type = "ECSServiceAverageCPUUtilization"
		}
		target_value = 60.0
	}
}
```

If you'd like, I can add `ecs-fargate/alb.tf` and `ecs-fargate/autoscaling.tf` with these snippets and adjust `main.tf` to use private subnets. Say the word and I'll scaffold them.
