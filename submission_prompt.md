You are a senior AWS cloud architect and DevSecOps engineer.
Design and generate a production-ready AWS implementation for this use case:

USE CASE
- A public website widget sends POST webhooks after successful token purchases.
- Payload includes buyer form fields, wallet address, tx hash, purchased amount, chainId, and timestamp.
- We need to:
  1) securely receive webhook payloads,
  2) validate and deduplicate requests,
  3) store events for audit/reporting,
  4) send email notifications to an owner inbox,
  5) expose an admin read API for querying purchases.

REQUIREMENTS
- AWS services: API Gateway, Lambda, DynamoDB, SES, CloudWatch, IAM, KMS, SSM Parameter Store.
- Infrastructure as code in Terraform (preferred), modularized.
- Region: eu-central-1 (parameterized).
- Security:
  - webhook HMAC signature verification (X-Signature header),
  - least-privilege IAM,
  - KMS encryption at rest where applicable,
  - API throttling/rate limiting,
  - strict input schema validation,
  - no secrets in code; use SSM Parameter Store secure strings.
- Reliability:
  - idempotency by txHash + buyerAddress + chainId,
  - retry-safe behavior,
  - structured logs with correlation IDs,
  - CloudWatch alarms for Lambda errors and API 5xx.
- Cost optimization:
  - serverless only,
  - DynamoDB on-demand,
  - log retention set to 14 days by default.
- Operations:
  - include runbook for deployment and rollback,
  - include smoke test commands and sample curl payloads.

DELIVERABLES
1) Architecture overview with data flow.
2) Terraform code structure with files and exact resource definitions.
3) Lambda handler code (Python 3.12) for:
   - signature verification,
   - payload validation,
   - dedup logic in DynamoDB,
   - SES email sending.
4) API contract:
   - POST /purchase-webhook
   - GET /purchases?wallet=...&from=...&to=...
5) Example payload schema and validation rules.
6) Step-by-step deploy instructions.
7) Troubleshooting section with common failures and fixes.
8) Security checklist and Well-Architected mapping.
9) Minimal integration snippet for a Wix/custom JS frontend showing webhook call pattern.

CONSTRAINTS
- Make all names/environment-specific values configurable (project, stage, region, email).
- Use clear comments and explain tradeoffs.
- Output should be directly usable by engineers, not high-level only.