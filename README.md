# Production-Ready On-Chain Purchase Notification Pipeline (AWS)

![Project Logo](aws_prompt_logo.jpg)

A production-ready, serverless blueprint for handling purchase webhooks from a public website widget and turning them into secure, reliable, and auditable events on AWS.

This repository contains the project context and submission assets for an AWS prompt challenge. The core prompt is designed to generate a full implementation with Terraform and Lambda for secure webhook ingestion, deduplication, storage, and email notifications.

## Problem

Public purchase webhooks are easy to start but hard to run safely in production. Teams typically face:

- Unverified inbound requests and weak secret handling
- Duplicate/replayed events from network retries
- Missing audit trail and poor event queryability
- Fragile notification workflows
- Limited visibility into failures and operational health

## Solution Overview

The prompt-driven solution builds a serverless AWS pipeline that:

1. Receives webhooks through Amazon API Gateway
2. Verifies webhook HMAC signatures from `X-Signature`
3. Validates payload schema and required fields
4. Deduplicates events using an idempotency key
5. Persists records in Amazon DynamoDB
6. Sends owner notifications through Amazon SES
7. Exposes an admin read API for purchase lookups
8. Monitors health with Amazon CloudWatch logs and alarms

## Architecture

### Data Flow

1. Website widget sends `POST /purchase-webhook`
2. API Gateway forwards request to Lambda handler
3. Lambda:
   - pulls webhook secret from AWS Systems Manager Parameter Store
   - verifies signature
   - validates payload schema
   - checks and writes idempotent record in DynamoDB
   - sends notification email via SES
4. Admin user calls `GET /purchases?wallet=...&from=...&to=...`
5. API returns filtered purchase records

### AWS Services Used

- Amazon API Gateway
- AWS Lambda (Python 3.12)
- Amazon DynamoDB (on-demand)
- Amazon SES
- Amazon CloudWatch
- AWS Identity and Access Management (IAM)
- AWS Key Management Service (KMS)
- AWS Systems Manager Parameter Store (SecureString)

## Repository Contents

- `submission_prompt.md`: complete prompt used for implementation generation
- `image_prompt.md`: prompt used for logo/image generation
- `aws_prompt_logo.jpg`: generated logo asset

## API Contract

### `POST /purchase-webhook`

Receives purchase webhook payloads from a public widget.

Expected high-level fields:

- buyer form fields
- `walletAddress`
- `txHash`
- `purchasedAmount`
- `chainId`
- `timestamp`

Required behavior:

- HMAC signature verification via `X-Signature`
- strict schema validation
- idempotency key based on `txHash + buyerAddress + chainId`
- durable write to DynamoDB
- owner email notification via SES

### `GET /purchases?wallet=...&from=...&to=...`

Returns purchase records for admin/ops workflows.

## Security Controls

- HMAC verification for inbound webhooks
- IAM least-privilege policies per component
- KMS encryption at rest where applicable
- Secrets stored in SSM Parameter Store (no secrets in code)
- API throttling/rate limiting at API Gateway
- Input schema validation to reduce injection and malformed payload risks

## Reliability and Operations

- Retry-safe processing with idempotency enforcement
- Structured logs and correlation IDs
- CloudWatch alarms for Lambda errors and API 5xx
- Runbook-driven deployment and rollback process
- Smoke tests for API, storage, and notifications

## Cost Controls

- Serverless architecture only (no always-on compute)
- DynamoDB on-demand billing mode
- CloudWatch log retention set to 14 days by default

## Configuration

All environment-specific values should be parameterized:

- `project`
- `stage` (dev/staging/prod)
- `region` (default target: `eu-central-1`)
- owner notification email
- webhook secret parameter path

## Quick Start (Implementation Runbook)

1. Configure AWS credentials and target account.
2. Create/store webhook secret in SSM Parameter Store as `SecureString`.
3. Verify SES sender identity/domain.
4. Set Terraform variables (`project`, `stage`, `region`, `owner_email`).
5. Deploy infrastructure with Terraform modules.
6. Run smoke tests against `POST /purchase-webhook`.
7. Verify:
   - DynamoDB record creation
   - SES email delivery
   - CloudWatch logs/metrics/alarms

## Smoke Test Example (Conceptual)

Send a signed webhook payload to the ingestion endpoint, then:

- assert HTTP success code
- verify a single DynamoDB record for idempotency key
- re-send same payload and verify no duplicate record
- confirm notification email behavior

## Troubleshooting

- **Signature mismatch:** verify canonical payload generation and shared secret source in SSM.
- **No emails sent:** check SES sandbox status, verified identities, suppression list, and Lambda permissions.
- **Unexpected duplicates:** validate idempotency key composition and conditional write logic.
- **5xx spikes:** inspect CloudWatch logs for Lambda exceptions and dependency failures.

## AWS Well-Architected Mapping

- **Security:** signature verification, least privilege, encryption, secret management
- **Reliability:** idempotency, retries, alarms, observable failure modes
- **Performance Efficiency:** serverless scaling and lightweight event path
- **Cost Optimization:** on-demand services and controlled log retention
- **Operational Excellence:** IaC, repeatable runbook, smoke tests, clear troubleshooting

## Intended Audience

- Startup engineering teams receiving purchase webhooks
- DevOps/Platform engineers building secure event pipelines
- Builders needing production-ready AWS prompt patterns

## License and Usage

Use this repository content as a reference implementation and submission artifact. Adapt service limits, governance controls, and data policies to your organization requirements before production use.
