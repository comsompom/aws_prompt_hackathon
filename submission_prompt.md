You are a senior AWS cloud architect and DevSecOps engineer with production experience
building serverless event ingestion pipelines. Generate a complete, directly-runnable
implementation. Every file must be complete — no ellipsis, no "add your logic here"
placeholders.

CONTEXT

A public website widget fires a POST webhook after each on-chain token purchase.
The payload is untrusted; the sender is anonymous. Volume is low (< 1000/day) but
each event is financially significant — duplicates and missed notifications are
unacceptable.

SUBMISSION GOAL

The generated repository must be good enough for two purposes at once:
1. a developer can deploy it with minimal adaptation;
2. it reads like a high-quality AWS prompt challenge submission with clear
   prerequisites, use case, expected outcome, AWS service mapping, security
   controls, cost controls, and troubleshooting guidance.

EXACT PAYLOAD CONTRACT  ← pin this; do not invent fields

POST body (application/json):
{
  "buyerEmail":        string, valid email, required
  "buyerName":        string, 1-100 chars, required
  "walletAddress":    string, EIP-55 checksum address (0x + 40 hex), required
  "txHash":           string, 0x + 64 hex, required
  "purchasedAmount":  string, decimal number > 0 as string (avoid float loss), required
  "chainId":          integer, one of [1, 56, 137, 42161], required
  "timestamp":        string, ISO-8601 UTC, required
}

Header: X-Signature: hmac-sha256=<hex_digest>
Canonical body for HMAC: the raw request body bytes exactly as received.

Idempotency key:  sha256(txHash + ":" + walletAddress.lower() + ":" + str(chainId))

DYNAMODB SCHEMA  ← specify exactly; do not let the model invent this

Table name: var.project-var.stage-purchases

Primary key
  PK  (S): "PURCHASE"          — single entity type, allows future expansion
  SK  (S): <idempotency_key>   — sha256 hex, 64 chars

Attributes written on first insert:
  idempotency_key  (S)
  wallet_address   (S)
  tx_hash          (S)
  chain_id         (N)
  purchased_amount (S)
  buyer_email      (S)
  buyer_name       (S)
  timestamp        (S)  — ISO-8601 from payload
  received_at      (S)  — Lambda system time, ISO-8601
  correlation_id   (S)  — uuid4 assigned per invocation
  status           (S)  — "RECEIVED" on insert; "NOTIFIED" after SES success
  ttl              (N)  — epoch seconds, now + 365 days (enables future expiry)

GSI-1  (for the admin read API):
  GSI PK  wallet_gsi   (S): wallet_address
  GSI SK  received_at  (S): ISO-8601 (lexicographic sort works for ISO dates)
  Projection: ALL
  Name: wallet-received-index

Conditional write:
  condition_expression="attribute_not_exists(PK)"
  — raises ConditionalCheckFailedException on duplicate; treat as 200 OK + log

Billing: PAY_PER_REQUEST
Encryption: aws_managed_key (SSE-KMS, use the default DynamoDB-owned key unless
            var.use_cmk=true, in which case create a customer KMS key)

TERRAFORM MODULE STRUCTURE  ← exact file tree required

infra/
  main.tf                — provider config, partial backend block (S3+DynamoDB lock),
                           module calls
  variables.tf           — project, stage, region, owner_email,
                           webhook_secret_ssm_path, use_cmk, alarm_email,
                           admin_cors_origin, ses_identity_type, ses_identity_value
  outputs.tf             — api_gateway_url, table_name, log_group_name
  terraform.tfvars.example
  backend.hcl.example    — bucket, key, region, dynamodb_table placeholders used by
                           terraform init -backend-config=backend.hcl

  modules/
    api_gateway/
      main.tf            — REST API, stage, deployment, POST+GET resources,
                           throttle: burst=50 rate=20, usage plan
      variables.tf
      outputs.tf         — invoke_url, api_id

    lambda/
      main.tf            — two functions: webhook_handler + purchases_reader
                           runtime=python3.12, arch=arm64 (Graviton, cheaper)
                           memory=256MB timeout=15s, reserved_concurrency=10
                           environment variables: TABLE_NAME, GSI_NAME,
                           SECRET_SSM_PATH, OWNER_EMAIL, POWERTOOLS_SERVICE_NAME,
                           LOG_LEVEL
                           source_code_hash triggers on zip change
      iam.tf             — per-function roles; webhook role: ssm:GetParameter,
                           dynamodb:PutItem, ses:SendEmail, kms:Decrypt (if CMK),
                           logs:CreateLogGroup/Stream/PutLogEvents;
                           reader role: dynamodb:Query on GSI only
      variables.tf
      outputs.tf         — webhook_function_arn, reader_function_arn,
                           webhook_function_name

    dynamodb/
      main.tf            — table + GSI, TTL attribute=ttl, PITR=true,
                           KMS block conditional on var.use_cmk
      kms.tf             — conditional CMK + key policy (only when use_cmk=true)
      variables.tf
      outputs.tf         — table_name, table_arn

    ssm/
      main.tf            — aws_ssm_parameter data source (do NOT create the secret
                           here — it must exist before deploy; just validate and
                           export the ARN for IAM)
      outputs.tf         — secret_arn

    ses/
      main.tf            — aws_ses_domain_identity or aws_ses_email_identity
                           conditional on var.ses_identity_type ("email"|"domain")
                           Output a warning if sandbox mode cannot be detected
      variables.tf
      outputs.tf

    monitoring/
      main.tf            — CloudWatch log groups (retention=14 days), metric filters,
                           two alarms: lambda_errors (>0 in 5min) and
                           api_5xx (>2 in 5min), SNS topic for alarm_email,
                           dashboard with 4 widgets: invocations, errors,
                           p99 duration, DynamoDB consumed write units
      variables.tf
      outputs.tf         — dashboard_url, alarm_topic_arn

Lambda source lives outside infra/:
  src/
    webhook_handler/
      handler.py
      requirements.txt   — aws-lambda-powertools>=2.30, boto3 (provided by runtime),
                           jsonschema>=4.21
    purchases_reader/
      handler.py
      requirements.txt   — aws-lambda-powertools>=2.30
  scripts/
    build_lambda.sh      — builds zips for both functions into infra/lambda_zips/
    smoke_test.sh        — full smoke test with signed curl calls

DOCUMENTATION ARTIFACTS  ← required for challenge-quality output

Generate:
  README.md
    Must include these sections exactly:
      - Problem / Use Case
      - Intended Audience
      - Prerequisites
      - Expected Outcome
      - Architecture Overview
      - AWS Services Used
      - Security Controls
      - Reliability and Operations
      - Cost Controls
      - Deployment Steps
      - Smoke Test Summary
      - AWS Well-Architected Mapping
  TROUBLESHOOTING.md
    Must include at least 8 realistic failure modes with symptom, likely cause,
    and corrective action.

LAMBDA IMPLEMENTATION REQUIREMENTS

Use aws-lambda-powertools for:
  - @logger.inject_lambda_context  — auto-injects request_id + cold_start
  - @tracer.capture_lambda_handler — X-Ray tracing (passive mode, no extra cost)
  - structured JSON logging with correlation_id field

webhook_handler.py — ordered processing steps:

  1. Extract correlation_id = context.aws_request_id
  2. Parse body — return 400 if not valid JSON with message "invalid_json"
  3. Verify X-Signature:
       expected = "hmac-sha256=" + hmac.new(secret, raw_body, sha256).hexdigest()
       Use hmac.compare_digest — never == comparison (timing attack)
       Return 401 {"error":"invalid_signature"} on mismatch
       Cache secret in module-level variable (populated once per cold start from SSM
       using GetParameter with WithDecryption=True); catch SSMParameterNotFound and
       raise a 500 with log.critical — do not expose the SSM error to the caller
  4. Schema validation with jsonschema — return 422 with detailed field errors
       Validate walletAddress regex: ^0x[0-9a-fA-F]{40}$
       Validate txHash regex: ^0x[0-9a-fA-F]{64}$
       Validate purchasedAmount: Decimal(value) > 0, no exception
       Validate chainId in [1, 56, 137, 42161]
  5. Compute idempotency_key = sha256(txHash+":"+walletAddress.lower()+":"+str(chainId))
  6. DynamoDB conditional PutItem — on ConditionalCheckFailedException:
       log.info("duplicate_event", idempotency_key=..., tx_hash=...)
       return 200 {"status":"duplicate","idempotency_key":...}
  7. SES SendEmail — use HTML + plain-text alternative parts:
       Subject: "New purchase – {purchasedAmount} tokens from {buyerName}"
       HTML body must include: buyer name, wallet (first 8 + last 6 chars),
       amount, chain name (map chainId: 1=Ethereum, 56=BSC, 137=Polygon,
       42161=Arbitrum), tx hash with Etherscan/explorer link, timestamp
       On SES failure: log.error and update DynamoDB status to "NOTIFY_FAILED"
       — do NOT raise (already persisted; notification failure must not cause retry)
  8. Update DynamoDB item status to "NOTIFIED" with UpdateItem
  9. Return 200 {"status":"ok","idempotency_key":...,"correlation_id":...}

  All 4xx/5xx responses follow: {"error": "<snake_case_code>", "message": "<human>"}

purchases_reader.py:
  GET /purchases
  Query params: wallet (required), from (ISO date, optional), to (ISO date, optional)
  Validate wallet regex before querying.
  Use GSI wallet-received-index:
    KeyConditionExpression: wallet_gsi = :w AND received_at BETWEEN :from AND :to
    If from/to absent: default to last 30 days
  Return: {"purchases": [...], "count": N, "wallet": "..."}
  Max 100 items per call (Limit=100); add LastEvaluatedKey pagination if needed and
  include next_cursor in the response when more data exists.
  Strip buyer_email from response (PII — admin API should not expose it).

API GATEWAY SPECIFICS

REST API (not HTTP API) — required for usage plans + WAF attachment point.
Endpoint type: REGIONAL.
Stage name: var.stage.
Deploy automatically on every plan (lifecycle ignore_changes = [] — we accept
re-deployments; for zero-downtime canary deploys add a comment explaining how).

POST /purchase-webhook:
  Integration: Lambda proxy
  Request validator: validate body + headers
  Method request: Content-Type: application/json required
  Response models: 200, 400, 401, 422, 500 — each with explicit JSON schema model

GET /purchases:
  Integration: Lambda proxy
  Request validator: validate query string params
  Required query param: wallet
  Authorization: AWS_IAM
  Do not treat API keys as authentication. Add a code comment explaining that
  SigV4-signed requests are required because this is an admin data endpoint.

CORS:
  POST endpoint: allow origin *, allow headers Content-Type + X-Signature
  GET endpoint: restrict to var.admin_cors_origin (default "")

Throttle (default stage throttle):
  burst_limit = 50
  rate_limit  = 20

SECURITY CHECKLIST (generate each item as a code comment or inline in resources)

□ HMAC uses constant-time compare (hmac.compare_digest)
□ Secret never logged — SSM value masked in Powertools logger
□ walletAddress lowercased before idempotency key (case-insensitive dedup)
□ DynamoDB item never contains raw request body (field-level storage only)
□ Lambda reserved_concurrency=10 limits blast radius on traffic spike
□ SES sender must be a verified identity — Terraform outputs a reminder if unverified
□ IAM: reader Lambda cannot Write to DynamoDB (separate role, Query only)
□ IAM: webhook Lambda cannot Query/Scan DynamoDB (PutItem + UpdateItem only)
□ Log group KMS encryption if use_cmk=true
□ buyer_email stripped from GET /purchases response

ERROR HANDLING MATRIX (implement all cases explicitly)

Trigger                        | HTTP | DynamoDB write | SES | Log level
-------------------------------|------|----------------|-----|----------
Invalid JSON                   | 400  | No             | No  | WARNING
Signature mismatch             | 401  | No             | No  | WARNING
Schema validation failure      | 422  | No             | No  | WARNING
Duplicate (idempotency hit)    | 200  | No             | No  | INFO
SSM unavailable                | 500  | No             | No  | CRITICAL
DynamoDB write failure         | 500  | Failed         | No  | ERROR
SES send failure               | 200  | Update FAILED  | No  | ERROR
Unexpected exception           | 500  | Attempt update | No  | CRITICAL

BUILD & DEPLOY RUNBOOK (generate as scripts/deploy.sh)

Prerequisites check:
  - aws-cli v2, terraform >= 1.7, python 3.12, jq, openssl
  - AWS credentials with AdministratorAccess (or named policy list)

Steps (each as a numbered bash block with set -euo pipefail):
  1. Create SSM secret:
       aws ssm put-parameter --name "$SECRET_PATH" --value "$WEBHOOK_SECRET" \
         --type SecureString --region eu-central-1
  2. Verify SES identity (email or domain — show both variants)
  3. Build Lambda zips:
       cd src/webhook_handler && pip install -r requirements.txt -t package/
       cp handler.py package/ && cd package && zip -r ../../infra/lambda_zips/webhook.zip .
       (same pattern for purchases_reader)
  4. Create backend.hcl from backend.hcl.example, then run:
       terraform init -backend-config=backend.hcl
  5. terraform plan -var-file=terraform.tfvars -out=tfplan
  6. terraform apply tfplan
  7. Run smoke_test.sh (see below)
  8. Rollback:
       - preferred: re-apply the previous Terraform commit/state
       - emergency: aws lambda update-function-code --function-name ... --zip-file ...
     Do not mention image_tag or container rollback; this solution uses zip artifacts.

SMOKE TEST SCRIPT (scripts/smoke_test.sh)

Generate a complete bash script that:
  1. Reads WEBHOOK_SECRET and API_URL from env
  2. Constructs test payload as a shell variable
  3. Computes signature:
       SIG="hmac-sha256=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$WEBHOOK_SECRET" | awk '{print $2}')"
  4. Sends POST with curl -s -w "\n%{http_code}" — asserts 200
  5. Sends identical POST again — asserts 200 with status=duplicate
  6. Sends POST with wrong signature — asserts 401
  7. Sends POST with missing field — asserts 422
  8. Sends GET /purchases?wallet=<wallet> as a SigV4-signed request — asserts 200 and
     count >= 1; use curl --aws-sigv4 when available and include an AWS CLI fallback
  9. Checks DynamoDB directly:
       aws dynamodb get-item --table-name "$TABLE" \
         --key '{"PK":{"S":"PURCHASE"},"SK":{"S":"<idempotency_key>"}}'
  10. Prints PASS/FAIL for each step and exits 1 if any failed

FRONTEND SNIPPET

Generate a standalone TypeScript function (no framework dependency) that:
  - accepts the payload fields as typed parameters (define the interface)
  - POSTs to a configurable WEBHOOK_URL constant
  - does NOT compute the HMAC (that must stay server-side — explain why in a comment)
  - handles HTTP 200 (ok + duplicate both acceptable), 4xx (user-facing error),
    5xx (retry with exponential backoff, max 3 attempts, jitter)
  - is marked async and returns { success: boolean; error?: string }

Explain in a comment why HMAC signing must never happen in browser JS
(secret would be exposed in page source / devtools network tab).

TRADEOFFS TO DOCUMENT (inline comments in code, not a separate section)

Document these specific decisions where the relevant resource appears:
  - Why REST API not HTTP API (usage plans, WAF, request validators)
  - Why arm64 Lambda (20% cheaper, same performance for I/O-bound workload)
  - Why SSM GetParameter per cold start vs Secrets Manager (cost: $0 vs $0.05/10k)
  - Why reserved_concurrency=10 (blast-radius cap; explain the queue-depth tradeoff)
  - Why SHA-256 idempotency key not raw txHash (normalisation + fixed 64-char SK)
  - Why SES failure does NOT cause 500 (event is durable; retry storms worse than
    delayed notification)
  - Why buyer_email excluded from GET response (PII minimisation, GDPR)
  - Why DynamoDB TTL set even though no expiry needed yet (cheap; hard to add later)
  - Why GET /purchases uses AWS_IAM instead of API key auth (admin data, real auth)

TERRAFORM BACKEND RULE

Because Terraform backends cannot depend on normal input variables, generate:
  - a partial backend "s3" block in infra/main.tf with no hard-coded secrets
  - infra/backend.hcl.example with placeholder values
  - deploy.sh instructions that copy backend.hcl.example to backend.hcl and fill it in
Do not invent backend bucket names or pretend backend configuration can be driven from
terraform.tfvars.

OUTPUT FORMAT

Output each file as a fenced code block with the path as the filename comment
on the first line. Files in this order:
  1. Architecture diagram (ASCII, max 60 cols wide)
  2. README.md
  3. TROUBLESHOOTING.md
  4. infra/variables.tf
  5. infra/main.tf
  6. infra/outputs.tf
  7. infra/terraform.tfvars.example
  8. infra/backend.hcl.example
  9. infra/modules/dynamodb/main.tf
  10. infra/modules/dynamodb/kms.tf
  11. infra/modules/dynamodb/variables.tf
  12. infra/modules/dynamodb/outputs.tf
  13. infra/modules/lambda/main.tf
  14. infra/modules/lambda/iam.tf
  15. infra/modules/lambda/variables.tf
  16. infra/modules/lambda/outputs.tf
  17. infra/modules/api_gateway/main.tf
  18. infra/modules/api_gateway/variables.tf
  19. infra/modules/api_gateway/outputs.tf
  20. infra/modules/monitoring/main.tf
  21. infra/modules/monitoring/variables.tf
  22. infra/modules/monitoring/outputs.tf
  23. infra/modules/ssm/main.tf
  24. infra/modules/ssm/outputs.tf
  25. infra/modules/ses/main.tf
  26. infra/modules/ses/variables.tf
  27. infra/modules/ses/outputs.tf
  28. src/webhook_handler/handler.py
  29. src/webhook_handler/requirements.txt
  30. src/purchases_reader/handler.py
  31. src/purchases_reader/requirements.txt
  32. scripts/build_lambda.sh
  33. scripts/smoke_test.sh
  34. scripts/deploy.sh
  35. frontend/webhook-client.ts

Do not truncate any file. If a file would be very long, write it in full anyway.
