# Base CloudFormation Infrastructure

This repository contains modular AWS CloudFormation templates for provisioning the **base infrastructure** used by the Base backend across `dev`, `stg`, and `prod` environments.

It defines a network foundation, security boundaries, compute platform (ECS/Fargate), load balancing, data services, object storage, and supporting IAM roles.

## What This Repo Does

The templates in this repo are intended to be deployed as a stack set of foundational infrastructure components:

1. Build a multi-subnet VPC layout (public, web/app private, data private) with internet/NAT routing and VPC endpoints.
2. Create security groups for bastion, ALB, backend, and RDS data access.
3. Provision ECS cluster and backend service runtime resources.
4. Expose backend traffic through an ALB and target group.
5. Provision supporting data/storage services (RDS PostgreSQL, S3 bucket).
6. Create IAM roles needed by ECS tasks.

## Templates Overview

### `base-vpc-dev-stg.yaml`
Creates the network foundation:
- VPC with configurable CIDR and 2-3 AZ deployment.
- Public subnets, web/app private subnets, and data private subnets.
- Internet Gateway and route tables.
- Single NAT Gateway in AZ0, reused by private subnet route tables.
- VPC endpoints:
  - Gateway endpoint: S3
  - Interface endpoints: ECR (`api`, `dkr`), CloudWatch Logs, SSM, SSM Messages, EC2 Messages, Secrets Manager, KMS
- Security group for interface VPC endpoints.

Key outputs include VPC ID, subnet IDs (individual and comma-separated lists), endpoint SG, and S3 endpoint ID.

### `base-securitygroups.yaml`
Creates core security groups:
- `BastionSecurityGroup`: inbound SSH (`22`) from `SshAccessCidr`.
- `PublicAlbSecurityGroup`: inbound HTTP/HTTPS (`80/443`) from internet.
- `BackendSecurityGroup`: inbound app port `3000` from ALB SG only.
- `RDSSecurityGroup`: inbound PostgreSQL (`5432`) from backend and bastion SGs.

Outputs all SG IDs for cross-stack usage.

### `base-role.yaml`
Creates ECS task execution role:
- IAM role assumed by `ecs-tasks.amazonaws.com`.
- AWS managed policy `AmazonECSTaskExecutionRolePolicy`.
- Inline policy for SSM Parameter Store reads under `/${SystemName}/*`.

### `base-ecs-cluster.yaml`
Creates an ECS cluster:
- `AWS::ECS::Cluster` named `${SystemName}-Cluster-${Environment}`.
- Outputs cluster ID/reference.

### `base-alb-backend-services.yaml`
Creates public ALB resources for backend traffic:
- Internet-facing ALB.
- HTTP listener with direct forwarding when ACM certificate ARN is not provided.
- HTTP-to-HTTPS redirect and HTTPS listener when ACM certificate ARN is provided.
- IP target group for backend on port `3000` with `/health` check.

Outputs include ALB DNS name, hosted zone ID, full name, target group ARN, and a computed ALB hostname.

### `base-docker-service-Dev.yaml`
Creates ECS/Fargate backend service runtime:
- ECS task definition (Fargate, `awsvpc`, CloudWatch logs).
- Container secrets loaded from SSM Parameter Store paths.
- ECS service attached to ALB target group.
- Application Auto Scaling target + CPU/memory target tracking policies.
- CloudWatch alarms for high CPU and memory utilization.

### `base-rds.yaml`
Creates PostgreSQL RDS instance and DB networking artifacts:
- DB subnet group across 3 private subnets.
- DB parameter group (`postgres17`) with:
  - `rds.force_ssl=1`
  - custom `max_connections`
- Encrypted DB storage (`gp3`), private accessibility, 7-day backups.
- Snapshot retention behavior on delete/replace.
- Performance Insights enabled.
- Deletion protection enabled.

Outputs DB identifier, endpoint, and user.

### `base-s3-backend.yaml`
Creates backend S3 storage and scoped IAM access:
- S3 bucket `${paramBucketName}-${paramEnvironment}`.
- Public access block (all controls enabled).
- Versioning enabled.
- Server-side encryption enabled (`AES256`).
- Bucket policy denying non-TLS (`aws:SecureTransport=false`) access.
- IAM user + access key with scoped S3 object/list permissions.

Outputs bucket details and IAM access artifacts.

### `base-bastion.yaml`
Creates a bastion host for administrative access:
- EC2 instance in selected subnet with key pair.
- Bastion IAM role/profile for CloudWatch Logs permissions.
- IMDSv2 required.
- Encrypted root EBS volume.

Outputs instance ID and public/private IPs.

## Suggested Deployment Order

1. `base-vpc-dev-stg.yaml`
2. `base-securitygroups.yaml`
3. `base-role.yaml`
4. `base-ecs-cluster.yaml`
5. `base-alb-backend-services.yaml`
6. `base-docker-service-Dev.yaml`
7. `base-rds.yaml`
8. `base-s3-backend.yaml`
9. `base-bastion.yaml`

Use outputs from earlier stacks (VPC, subnets, SGs, target group, cluster) as parameters in later stacks.

## AWS Well-Architected Practices (Relevant to This Repo)

This section maps the templates to AWS Well-Architected guidance.

### Security
Practices already reflected:
- Network segmentation (public/web/data subnets).
- Least-privilege network paths through SG-to-SG rules.
- Encryption at rest for RDS and S3.
- TLS enforcement for S3 access.
- Secret retrieval via SSM Parameter Store.
- IMDSv2 required on bastion.

Improvements recommended:
- Replace IAM user access keys (`base-s3-backend.yaml`) with IAM roles and short-lived credentials.
- Avoid direct SSH bastion access where possible; prefer SSM Session Manager.
- Review hardcoded account/region-specific ARNs in ECS service template.

### Reliability
Practices already reflected:
- Multi-AZ subnet layout.
- ECS desired count and autoscaling policies.
- RDS backup retention + snapshot policies.

Improvements recommended:
- Single NAT Gateway is a cost optimization but an AZ-level SPOF for egress; consider one NAT per AZ for higher resilience.
- Consider Multi-AZ RDS for production workloads.

### Performance Efficiency
Practices already reflected:
- ECS autoscaling on CPU/memory.
- Private VPC endpoints reduce dependency on public egress for AWS services.

Improvements recommended:
- Validate right-sizing of ECS CPU/memory and RDS class per environment.
- Add load/performance testing feedback loop before capacity changes.

### Cost Optimization
Practices already reflected:
- Single NAT Gateway pattern lowers baseline cost.
- Environment-aware service sizing (`DesiredCount`, task resources).

Improvements recommended:
- Add lifecycle policies and access logging where appropriate (S3, logs).
- Use budget and anomaly detection alerts tied to each environment.

### Operational Excellence
Practices already reflected:
- Infrastructure as code with modular templates.
- CloudWatch logs and alarms for ECS service health signals.

Improvements recommended:
- Add CI/CD validation (`cfn-lint`, `cloudformation validate-template`, drift detection).
- Standardize stack parameter files per environment.
- Add tagging standards (`Owner`, `CostCenter`, `Environment`, `DataClassification`) across all resources.

## Cost Estimation (As-Is Templates)

Estimated monthly cost for deploying this repository **as currently written**, for one environment in `us-east-1`.

Assumptions used:
- 730 hours/month.
- On-Demand pricing.
- Minimal baseline traffic (no heavy data transfer).
- ACM certificate not included.
- RDS sizing/storage inputs are required parameters in `base-rds.yaml`, so exact DB monthly total depends on your selected values.

| Resource (template) | As-is configuration | Est. monthly (USD) |
|---|---|---:|
| NAT Gateway (`base-vpc-dev-stg.yaml`) | 1 NAT Gateway | 32.85 |
| Interface VPC Endpoints (`base-vpc-dev-stg.yaml`) | 8 endpoints (ECR API, ECR DKR, Logs, SSM, SSMMessages, EC2Messages, SecretsManager, KMS) | 58.40 |
| Application Load Balancer (`base-alb-backend-services.yaml`) | 1 ALB + low baseline LCU usage | 22.27 |
| ECS Fargate service (`base-docker-service-Dev.yaml`) | 1 task running continuously, **1 vCPU + 2 GB RAM** | 36.04 |
| Bastion EC2 (`base-bastion.yaml`) | 1 x `t3.nano` | 3.80 |
| CloudWatch Alarms (`base-docker-service-Dev.yaml`) | 2 standard metric alarms | 0.20 |
| S3 bucket (`base-s3-backend.yaml`) | Bucket + IAM user/key | Variable (storage/requests) |
| RDS PostgreSQL (`base-rds.yaml`) | Single-AZ, class/storage defined at deploy time | Variable (instance/storage/backup) |

Estimated baseline subtotal (excluding variable S3/RDS/data transfer): **~153.56 USD/month**.
