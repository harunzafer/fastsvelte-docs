---
description: "Deploy FastSvelte on AWS - ECS/Fargate for containers, RDS for PostgreSQL, and S3/CloudFront for frontend."
keywords: "aws deployment, aws ecs, aws fargate, aws rds, fastsvelte aws"
---

# Deploy to AWS

Deploy FastSvelte on Amazon Web Services using ECS, RDS, and S3 for a fully-managed cloud solution.

## What You'll Use

- **ECS/Fargate** - Run your FastAPI backend container serverlessly
- **RDS PostgreSQL** - Managed PostgreSQL database
- **S3 + CloudFront** - Host and distribute your frontend globally

## Cost Estimate

For a small production app:
- Fargate (0.25 vCPU, 0.5GB): ~$15-20/month
- RDS (db.t3.micro): ~$15-25/month
- S3 + CloudFront: ~$5-10/month

**Total:** ~$35-55/month

Note: AWS pricing is complex. Consider using [Neon](neon.md) (~$20/month) for database to save costs.

## Why AWS?

- **Enterprise-grade** - Trusted by large organizations
- **Comprehensive** - Every service you could need
- **Global infrastructure** - Deploy anywhere
- **Mature ecosystem** - Lots of tooling and support

## Deployment Guide

**Coming soon!** This guide is under development.

AWS has extensive documentation:
- [ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [RDS PostgreSQL](https://docs.aws.amazon.com/rds/index.html)
- [S3 Static Website Hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)

## What You'll Need

1. **Backend:** Docker container on ECS Fargate
2. **Database:** RDS PostgreSQL instance (or use Neon/Supabase)
3. **Frontend:** S3 bucket with CloudFront CDN

## Key Features

- Highly scalable infrastructure
- Built-in monitoring (CloudWatch)
- Load balancing (ALB/NLB)
- Auto-scaling groups
- IAM for fine-grained access control
- VPC for network isolation

---

Want to contribute this guide? [Open a PR](https://github.com/harunzafer/fastsvelte) with your deployment experience!
