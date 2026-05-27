# W7 Evidence Pack — AI Document Hub

## 1. Cover
- Group ID: G5
- Members: Minh, Nam, Hoàng, Vinh, Sơn, Quyên, Thủy
- Live URL: [điền sau khi deploy]
- GitHub repo: [điền link repo]
- Demo video: [điền link]
- Total spend: $[điền sau]

---

## 2. Domain + Use Case
- Domain: C — ProductivityTech
- App name: AI Document Hub (DocHub)
- Target users: Legal teams, compliance officers, knowledge workers managing large document libraries
- User pain: Teams waste hours searching through contracts and policies to find specific clauses or obligations
- Real-world parallel: Harvey AI, Ironclad, Glean, Microsoft Copilot for M365

---

## 3. Architecture

### Final architecture diagram
![Architecture Diagram](../Architecture_diagram.png)

### Service mapping — 7 mandatory capabilities
| # | Capability | Service | Why |
|---|-----------|---------|-----|
| 1 | User-Facing Entry | CloudFront + S3 static frontend + ALB | Public HTTPS, low ops overhead |
| 2 | Application Compute | ECS Fargate + FastAPI in private subnets | Container handles PDF libraries better than Lambda |
| 3 | AI/ML Feature | Bedrock KB + Claude Haiku + Titan Embeddings | RAG with metadata filter for tenant isolation |
| 4 | Data Persistence | DynamoDB | Tenant/workspace/doc metadata is key-value access pattern |
| 5 | Object Storage | S3 | Stores uploaded contracts/reports/policies |
| 6 | Network Foundation | VPC + public subnets (ALB) + private subnets (ECS) + SG | Backend not public, only ALB can reach ECS |
| 7 | Identity & Access | IAM least-privilege ECS task role + execution role | No wildcard policies |

### Optional capability
- #8 Full Observability — CloudWatch dashboard + custom metric + alarm + Log Insights

### Data flow

User uploads PDF/TXT → CloudFront → ALB → ECS FastAPI → S3 PutObject (tenants/{tenant_id}/workspaces/{workspace_id}/raw/{doc_id}/v{n}/file) → DynamoDB PutItem (tenant_id, workspace_id, doc_id, version, is_latest=true) → Bedrock KB StartIngestionJob (metadata: tenant_id, workspace_id, doc_id, version, is_latest)

User asks question → CloudFront → ALB → ECS FastAPI → Bedrock retrieve_and_generate (filter: tenant_id=X AND workspace_id=Y AND is_latest=true) → Claude Haiku generates answer + citations [filename - version - chunk_id] → DynamoDB PutItem (query history) → Response to UI

---

## 4. Service Decisions and Trade-offs

### DECISION 1: ECS Fargate over Lambda

ALTERNATIVES CONSIDERED:
- Lambda + Mangum — cheaper, but risky for PDF dependencies, package size limits, and longer processing
- EC2 — flexible, but more ops overhead and patching in 48h

MEASUREMENT:
- Docker image size: [điền sau] MB
- Average /health latency through ALB: [điền sau] ms
- Average upload processing time: [điền sau] ms
- Estimated 48h ECS + ALB cost: $[điền sau]

EVIDENCE:
- Screenshot: docs/screenshots/day1/ecs_service_healthy.png
- Screenshot: docs/screenshots/day1/alb_health_check.png

TRADE-OFF ACCEPTED:
- Higher fixed cost than Lambda, accepted to reduce packaging/runtime risk for document processing

---

### DECISION 2: DynamoDB over RDS

ALTERNATIVES CONSIDERED:
- RDS PostgreSQL — better for relational analytics, unnecessary for simple metadata lookup
- DocumentDB — fits nested metadata, too expensive/slow to set up in 48h

MEASUREMENT:
- Access pattern 1: list docs by tenant/workspace — key-value lookup
- Access pattern 2: save query history by tenant/workspace — key-value lookup
- Access pattern 3: get latest version for doc_id — key-value lookup
- All patterns do not require JOIN

EVIDENCE:
- Screenshot: docs/screenshots/day1/dynamodb_items.png

TRADE-OFF ACCEPTED:
- Gave up SQL flexibility for faster setup, lower cost, simpler tenant partitioning

---

### DECISION 3: Bedrock KB with metadata filters over direct InvokeModel

ALTERNATIVES CONSIDERED:
- Direct InvokeModel only — fastest to build, cannot ground answers in uploaded docs
- Full custom vector DB — more control, too much implementation risk in 48h

MEASUREMENT:
- Probe queries tested: [điền sau]
- Wrong-document retrieval count: [điền sau] / [điền sau]
- Citation present: [điền sau] / [điền sau]
- Latest-version chosen correctly: [điền sau] / [điền sau]

EVIDENCE:
- Screenshot: docs/screenshots/day2/qa_citations.png
- Screenshot: docs/screenshots/day2/latest_version_test.png

TRADE-OFF ACCEPTED:
- Bedrock KB less customizable, but much faster to ship. Metadata filters reduce wrong-document risk.

---

### DECISION 4: Claude Haiku over Sonnet

ALTERNATIVES CONSIDERED:
- Claude Sonnet — better reasoning, 3x higher cost ($3/$15 vs $1/$5 per 1M tokens)
- Titan Text — cheaper, weaker answer quality on our sample documents

MEASUREMENT:
- Haiku: $1.00/1M input, $5.00/1M output
- Sonnet: $3.00/1M input, $15.00/1M output
- [Điền sau: Haiku answered X/Y probe questions correctly]

EVIDENCE:
- Screenshot: docs/screenshots/day2/bedrock_logs.png

TRADE-OFF ACCEPTED:
- Slightly weaker reasoning than Sonnet, accepted to reduce cost and keep demo fast

---

## 5. Deployment Evidence

### Public URL
![Public URL](screenshots/day1/public_url.png)
- URL: [điền]
- Loads in browser, no SSL errors

### CloudFront + S3 frontend
![CloudFront](screenshots/day1/cloudfront.png)

### ALB health check
![ALB](screenshots/day1/alb_health_check.png)
- Target group: healthy

### ECS service + task
![ECS Service](screenshots/day1/ecs_service_healthy.png)
- Cluster: [tên cluster]
- Service: running
- Task: RUNNING

### ECR image
![ECR](screenshots/day1/ecr_image.png)
- Repository: [tên repo]
- Image pushed successfully

### Lambda CloudWatch logs
![CloudWatch logs](screenshots/day1/cloudwatch_logs.png)

### S3 object after upload
![S3 object](screenshots/day1/s3_object.png)
- Bucket: dochub-docs-g5-[account-id]
- Block Public Access: ON
- Versioning: ON
- Encryption: ON
- Path: tenants/tenant-a/workspaces/legal-contracts/raw/

### DynamoDB record
![DynamoDB](screenshots/day1/dynamodb_items.png)
- Table: DocHubMain
- PK: TENANT#tenant-a
- SK: DOC#legal-contracts#contract-001
- Fields: tenant_id, workspace_id, doc_id, version, is_latest=true

### IAM ECS task role
![IAM role](screenshots/day1/iam_role.png)
- No AdministratorAccess
- No wildcard Action: *

### VPC + Security Groups
![VPC](screenshots/day1/vpc_subnets.png)
- Public subnets: ALB only
- Private subnets: ECS tasks
![Security Groups](screenshots/day1/security_groups.png)
- ECS SG: inbound only from ALB SG

---

## 6. AI Evidence

### Sample input files
- File 1: docs/sample_input/contract-a-v1.txt
- File 2: docs/sample_input/contract-a-v2.txt
- File 3: docs/sample_input/contract-b.txt

### Summary output
![Summary](screenshots/day2/summary_output.png)
- Includes: executive summary, key obligations, important dates, risks

### Q&A with citation
![Q&A citation](screenshots/day2/qa_citations.png)
- Citation format: [filename - version - chunk_id]

### Search / clause check output
![Search](screenshots/day2/search_output.png)
- Cross-document search result with citations

### Tenant isolation evidence
![Tenant isolation](screenshots/day2/tenant_isolation.png)
- tenant-b list docs → cannot see tenant-a documents

### Latest version evidence
![Latest version](screenshots/day2/latest_version_test.png)
- Upload v2 → query → AI uses v2, not v1

### Retrieval evidence
- Chunking strategy: [điền — semantic vs fixed, chunk size]
- Metadata filters used: tenant_id, workspace_id, is_latest=true
- Wrong-document failure mode found: [điền — tên query bị sai, đã fix thế nào]
- Stale-version mitigation: [điền — is_latest field + DynamoDB latest_version]

---

## 7. Security and IAM

### ECS task role permissions
- s3:PutObject, s3:GetObject, s3:ListBucket
- dynamodb:PutItem, dynamodb:GetItem, dynamodb:Query, dynamodb:UpdateItem
- bedrock:InvokeModel
- bedrock-agent-runtime:Retrieve, RetrieveAndGenerate, InvokeAgent
- cloudwatch:PutMetricData

### ECS execution role permissions
- ecr:GetAuthorizationToken, BatchCheckLayerAvailability, GetDownloadUrlForLayer, BatchGetImage
- logs:CreateLogStream, PutLogEvents

### S3 security
- Block Public Access: ON
- Versioning: ON
- Default encryption: ON

### Network isolation
- ECS tasks in private subnets
- Only ALB SG can reach ECS SG
- S3/DynamoDB via Gateway Endpoints (free, no NAT Gateway)
- Bedrock via Interface Endpoint

---

## 8. Observability

### CloudWatch dashboard
![Dashboard](screenshots/day2/cloudwatch_dashboard.png)
- Widgets: ECS CPU/memory, ALB 4xx/5xx, DocHub/DocumentsUploaded

### Custom metrics
- DocHub/DocumentsUploaded
- DocHub/QueryLatencyMs
- DocHub/TenantFilteredQueries
- Published via PutMetricData from ECS backend

### Alarm
![Alarm](screenshots/day2/cloudwatch_alarm.png)
- Alarm on ECS error rate or custom metric
- State: OK (not INSUFFICIENT_DATA)

### Log Insights query
- Query: filter ERROR, Exception, AccessDenied, tenant_id, doc_id — sort desc — limit 50
![Log Insights](screenshots/day2/log_insights.png)

---

## 9. Cost

### Day 1 EOD screenshot
![Cost Day 1](screenshots/day1/cost_day1_eod.png)

### Day 2 EOD screenshot
![Cost Day 2](screenshots/day2/cost_day2_eod.png)

### Demo morning screenshot
![Cost Demo](screenshots/day2/cost_demo_morning.png)

### Top 3 cost drivers
1. OpenSearch Serverless — ~$27.65 (2 OCU × $0.288/hr × 48h) — vector store cho Bedrock KB
2. ECS Fargate + ALB — ~$[điền] — container compute + load balancer
3. Bedrock (Haiku + Titan Embeddings) — ~$1-2 — AI inference và document embedding

### Cost control notes
- Used Claude Haiku not Sonnet in dev
- No NAT Gateway — used VPC Gateway Endpoints for S3/DynamoDB
- ECS task count = 1 (minimum), small CPU/memory
- Deleted unused resources daily
- Delete OpenSearch Serverless collection immediately after demo
- Total spend: $[điền]

---

## 10. Lessons Learned

[Điền cuối Day 2 — khoảng 200 chữ]

Gợi ý:
- Điều gì hoạt động tốt?
- Điều gì thất bại hoặc mất nhiều thời gian hơn dự kiến?
- Nếu làm lại, sẽ thay đổi gì?
- Nếu Harvey AI engineer review architecture này, họ sẽ nói gì về wrong-document handling?

---

## 11. Teardown Plan

Delete resources theo thứ tự:
1. CloudFront — disable trước, rồi delete
2. ALB listener/rules — delete
3. ECS service — set desired count = 0, rồi delete service
4. ECS task definition — deregister
5. ECR image/repository — delete
6. ALB + target group — delete
7. Bedrock KB / OpenSearch Serverless — delete ngay sau demo
8. DynamoDB table — delete
9. S3 buckets — empty trước, rồi delete
10. CloudWatch dashboards/alarms/log groups — delete
11. IAM roles/policies — delete
12. VPC endpoints, subnets, route tables, SGs, VPC — delete cuối cùng

Teardown confirmation file: docs/teardown_confirmation.md
- List resource đã xóa
- Final Cost Explorer screenshot (chụp ngày 2/6)
- Tên người verify teardown
