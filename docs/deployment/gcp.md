---
description: "Deploy FastSvelte on Google Cloud Platform - Cloud Run for containers, Cloud SQL for PostgreSQL, and Cloud Storage for frontend."
keywords: "gcp deployment, google cloud, cloud run, cloud sql, fastsvelte gcp"
---

# Deploy to Google Cloud Platform

Deploy FastSvelte on GCP using Cloud Run, Cloud SQL, and Cloud Storage for a modern serverless architecture.

## What You'll Use

- **Cloud Run** - Run your FastAPI backend container serverlessly
- **Cloud SQL (PostgreSQL)** - Managed PostgreSQL database
- **Cloud Storage + Cloud CDN** - Host and distribute your frontend

## Cost Estimate

For a small production app:
- Cloud Run: ~$5-15/month (scales to zero)
- Cloud SQL (db-f1-micro): ~$10-20/month
- Cloud Storage + CDN: ~$5/month

**Total:** ~$20-40/month

Consider using [Neon](neon.md) or [Supabase](supabase.md) for more affordable database options.

## Why Google Cloud?

- **Serverless** - Cloud Run scales to zero
- **Modern** - Great developer experience
- **Firebase integration** - Easy auth and features
- **Generous free tier** - Good for getting started

## Deployment Guide

**Coming soon!** This guide is under development.

Google Cloud documentation:
- [Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Cloud SQL for PostgreSQL](https://cloud.google.com/sql/docs/postgres)
- [Cloud Storage Static Website](https://cloud.google.com/storage/docs/hosting-static-website)

## What You'll Need

1. **Backend:** Docker container on Cloud Run
2. **Database:** Cloud SQL PostgreSQL instance (or external provider)
3. **Frontend:** Cloud Storage bucket with Cloud CDN

## Key Features

- True serverless - pay only when running
- Automatic HTTPS
- Global load balancing
- Built-in monitoring (Cloud Monitoring)
- IAM for access control
- VPC connector for private networking

---

Want to contribute this guide? [Open a PR](https://github.com/harunzafer/fastsvelte) with your deployment experience!
