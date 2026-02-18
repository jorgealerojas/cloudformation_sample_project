# Repository Guidelines

## Project Structure & Module Organization
This repository is a modular AWS CloudFormation baseline. Each root-level template provisions one infrastructure domain:
- `base-vpc-dev-stg.yaml`: VPC, subnets, routing, endpoints.
- `base-securitygroups.yaml`: SG boundaries for ALB, backend, bastion, data.
- `base-role.yaml`, `base-ecs-cluster.yaml`: IAM role and ECS cluster.
- `base-alb-backend-services.yaml`, `base-docker-service-Dev.yaml`: ALB + ECS/Fargate service.
- `base-rds.yaml`, `base-s3-backend.yaml`, `base-bastion.yaml`: data/storage/admin access.

Keep templates independent and reusable; prefer cross-stack outputs/parameters over hardcoded values.

## Build, Test, and Development Commands
There is no app build step in this repo. Validate templates before opening a PR:
- `cfn-lint *.yaml` - static linting for CloudFormation best practices.
- `aws cloudformation validate-template --template-body file://base-vpc-dev-stg.yaml` - AWS parser validation (repeat per template).
- `aws cloudformation deploy --template-file base-securitygroups.yaml --stack-name <name> --parameter-overrides ... --capabilities CAPABILITY_NAMED_IAM` - deploy a stack manually.

Follow the deployment sequence documented in `README.md` to satisfy dependencies.

## Coding Style & Naming Conventions
Use YAML with 2-space indentation, descriptive logical IDs, and consistent parameter/output naming.
- Parameters: PascalCase (`DeploymentEnv`, `SystemName`).
- Resource logical IDs: PascalCase by domain (`BackendService`, `VpcEndpointSecurityGroup`).
- File names: `base-<domain>.yaml` pattern.

Prefer `!Sub`/`!Ref` over duplicated literals. Avoid embedding account-specific ARNs unless required.

## Testing Guidelines
Treat validation as required testing:
1. Run `cfn-lint *.yaml`.
2. Run `aws cloudformation validate-template` for changed files.
3. For behavioral changes, deploy to a `dev` stack and verify key outputs (VPC IDs, SG IDs, ALB DNS, DB endpoint).

## Commit & Pull Request Guidelines
Recent history uses short, imperative-style messages (`refactor adjustments`, `Adding sample templates`). Keep commits focused and descriptive:
- Good: `add rds parameter constraints`
- Avoid: `misc updates`

PRs should include:
- Purpose and impacted templates.
- Parameter/output changes and migration notes.
- Validation evidence (lint + validate-template commands run).
- Rollback considerations for stateful resources (RDS/S3).
