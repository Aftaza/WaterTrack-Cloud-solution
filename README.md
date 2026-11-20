# WaterTrack: Water Billing on AWS

Final project for the Cloud Architecture and Security course at Universitas Brawijaya (2026), built by a team of
five. WaterTrack is a water billing system, a Laravel API and a React SPA, deployed to AWS in Jakarta
(ap-southeast-3) the way a production system would be: highly available, private by default, and deployed from
GitHub without stored AWS keys.

## The app

| Role | What they do |
| --- | --- |
| **Admin** | Users, tariffs, customers, bills, payments, dashboard, audit log |
| **Operator** | Record meter readings (one by one or in bulk from an Excel template), generate bills, manage customer accounts |
| **Kasir** (cashier) | Look up and pay bills, confirm payment requests from customers |
| **Klien** (customer) | See their own bills, request payment, download a bill as PDF; must change the initial password first |

- Customers can be imported from CSV; subscription numbers are generated.
- Model observers write changes to users, customers, bills and payments to an audit log.
- Login is rate limited, and each route group is guarded by role middleware.

## AWS architecture

```
GitHub ──► GitHub Actions (OIDC) ──► ECR ──► ECS Fargate
                    └──────────────► S3 (SPA) + CloudFront invalidation

Users ─► Route 53 ─► CloudFront + WAF ─┬─► S3 (React SPA)
                                       └─► ALB ─► ECS Fargate (nginx + php-fpm, ≥ 2 tasks, auto scaling)
                                                     ├─► RDS MySQL Multi-AZ
                                                     ├─► ElastiCache Redis (TLS)
                                                     ├─► Secrets Manager
                                                     └─► S3 (uploads)
```

- ECS tasks, RDS and Redis live in private subnets; security groups only allow ALB → ECS → database and cache.
- Secrets Manager holds the app key and database credentials, injected at container start by the execution role.
  The task role reaches the uploads bucket, so the app holds no AWS keys.
- Two ACM certificates: one in ap-southeast-3 for the ALB, one in us-east-1 for CloudFront.
- CloudWatch logs and alarms (to SNS), X-Ray tracing, and CloudTrail.
- Estimated cost: about $138/month for production, about $70/month for a single-task staging setup.

Why ECS Fargate over EKS or plain EC2, and the rest of the decisions, are in
[`docs/aws-architecture/watertrack-aws-architecture.md`](docs/aws-architecture/watertrack-aws-architecture.md)
(diagram: [`.drawio`](docs/aws-architecture/watertrack-aws-architecture.drawio)).

## CI/CD

[`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) runs on push to `main` and only deploys the part that
changed:

- **Backend:** assume an IAM role through GitHub OIDC, build the image, push it to ECR tagged with the commit SHA,
  render a new task definition, and roll it out on ECS.
- **Frontend:** build the SPA, sync it to S3, and invalidate CloudFront.

## Repository

| Path | Stack |
| --- | --- |
| [`backend/`](backend) | Laravel 12, PHP 8.2+, Sanctum, MySQL, Redis (phpredis); nginx + php-fpm under supervisord |
| [`frontend/`](frontend) | React 18, TypeScript, Vite, Tailwind CSS, shadcn/ui |
| [`docs/`](docs) | Architecture, and the step-by-step deploy guides the team followed (Indonesian) |

## Running locally

```bash
docker compose up -d --build
docker compose exec backend php artisan migrate:fresh --seed --force
```

The SPA runs on http://localhost:3000 and the API on http://localhost:8000. The seeder creates
`admin@watertrack.id` and `operator@watertrack.id` with the password `password` (local only).

Backend tests: `docker compose exec backend php artisan test`.

## Deploying

- [`docs/deploy-guide-console.md`](docs/deploy-guide-console.md): AWS Console walkthrough (IAM, VPC, RDS, ElastiCache, ECS, CloudFront, WAF, CI/CD), split across five developers.
- [`docs/deploy-guide.md`](docs/deploy-guide.md): the same setup with the AWS CLI.

Account IDs and resource IDs in the docs are placeholders.

## Team

Forked from [johnnnic/WaterTrack](https://github.com/johnnnic/WaterTrack). My part: the AWS architecture and deploy
guides, the GitHub Actions pipeline, and most of the backend and frontend redesign.
