# W7 Evidence Pack — AI Study Buddy

## 1. Cover
- Group ID: G5
- Members: Minh, Nam, Hoàng, Vinh, Sơn, Quyên, Thủy
- Live URL: [điền sau khi deploy]
- GitHub repo: [điền link repo]
- Demo video: [điền link]
- Total spend: $[điền sau]

---

## 2. Domain + Use Case
- Domain: A — EduTech
- App name: AI Study Buddy (StudyBot)
- Target users: University students, self-learners, exam-prep candidates
- User pain: Students waste hours re-reading lecture notes to find key concepts
- Real-world parallel: Google NotebookLM, Quizlet AI, Khanmigo

---

## 3. Architecture

### Final architecture diagram
![Architecture Diagram](screenshots/day1/Architecture_diagram.png)

### Service mapping — 7 mandatory capabilities
| # | Capability | Service | Why |
|---|-----------|---------|-----|
| 1 | User-Facing Entry | CloudFront + S3 + API Gateway HTTP API | Public HTTPS, low ops overhead |
| 2 | Application Compute | Lambda + FastAPI + Mangum | Serverless, cheap, fast deploy |
| 3 | AI/ML Feature | Bedrock KB + Claude Haiku + Titan Embeddings | RAG grounds answers in uploaded docs |
| 4 | Data Persistence | DynamoDB | Key-value access pattern, no JOIN needed |
| 5 | Object Storage | S3 | Stores uploaded files + static frontend |
| 6 | Network Foundation | VPC + private subnets + Security Groups | DB not public |
| 7 | Identity & Access | IAM least-privilege execution roles | No wildcard policies |

### Optional capability
- #8 Full Observability — CloudWatch dashboard + custom metric + alarm + Log Insights

### Data flow
User uploads PDF → API Gateway POST /upload → Lambda → S3 PutObject → Bedrock KB StartIngestionJob → chunk + embed → Open Search Serverless

User asks question → API Gateway POST /query → Lambda → Bedrock retrieve_and_generate (filter by user_id) → Claude Haiku generates answer + citations → DynamoDB log → UI
---

## 4. Service Decisions and Trade-offs

### DECISION 1: DynamoDB over RDS

ALTERNATIVES CONSIDERED:
- RDS PostgreSQL — better for relational analytics, unnecessary for simple topic/doc lookup
- SQLite — easy local dev, not suitable for persistent cloud demo

MEASUREMENT:
- Access pattern 1: list docs by user/topic — single key lookup
- Access pattern 2: save query history — single key lookup
- Both patterns do not require JOIN

EVIDENCE:
- Screenshot: docs/screenshots/day1/dynamodb_items.png

TRADE-OFF ACCEPTED:
- Gave up SQL aggregation for faster setup, lower cost, simpler scaling

---

### DECISION 2: Bedrock KB (RAG) over direct InvokeModel

ALTERNATIVES CONSIDERED:
- Direct InvokeModel only — fastest to build, cannot ground answers in uploaded docs
- Full custom vector DB — more control, too much risk in 48h

MEASUREMENT:
- [Điền sau: X/5 probe questions answered correctly with citation]
- [Điền sau: average latency ___ ms]

EVIDENCE:
- Screenshot: docs/screenshots/day2/qa_citations.png

TRADE-OFF ACCEPTED:
- Bedrock KB less customizable, much faster to ship in 48h

---

### DECISION 3: Claude Haiku over Sonnet

ALTERNATIVES CONSIDERED:
- Claude Sonnet — better reasoning, 3x higher cost ($3/$15 vs $1/$5 per 1M tokens)
- Titan Text — cheaper, weaker answer quality on our sample questions

MEASUREMENT:
- Haiku: $1.00/1M input, $5.00/1M output
- Sonnet: $3.00/1M input, $15.00/1M output
- [Điền sau: Haiku answered X/5 probe questions correctly]

EVIDENCE:
- Screenshot: docs/screenshots/day2/bedrock_logs.png

TRADE-OFF ACCEPTED:
- Slightly weaker reasoning than Sonnet, accepted to reduce cost and keep demo fast

### DECISION 4: OpenSearch Serverless over S3 Vectors

ALTERNATIVES CONSIDERED:
- S3 Vectors — rẻ hơn (~$0.01/48h), nhưng availability tại
  ap-southeast-1 chưa confirm tại thời điểm build
- Self-managed vector DB — quá nhiều ops risk trong 48h

MEASUREMENT:
- OpenSearch Serverless: 2 OCU × $0.288/hr × 48h = ~$27.65
- S3 Vectors: ~$0.01 nếu available
- Budget còn lại sau OpenSearch: ~$72 — vẫn trong $100 cap

EVIDENCE:
- Screenshot: docs/screenshots/day1/opensearch_collection.png

TRADE-OFF ACCEPTED:
- Chi phí cao hơn S3 Vectors, nhưng đảm bảo availability
  và tích hợp native với Bedrock KB

---

## 5. Deployment Evidence

### Public URL
![Public URL](screenshots/day1/public_url.png)
- URL: [điền]
- Loads in browser, no SSL errors

### API Gateway
![API Gateway](screenshots/day1/api_gateway.png)

### Lambda function
![Lambda](screenshots/day1/lambda_function.png)

### Lambda CloudWatch logs
![CloudWatch logs](screenshots/day1/cloudwatch_logs.png)

### S3 object after upload
![S3 object](screenshots/day1/s3_object.png)
- Bucket: studybot-docs-g5-[account-id]
- Block Public Access: ON
- Versioning: ON
- Encryption: ON

### DynamoDB record
![DynamoDB](screenshots/day1/dynamodb_items.png)
- Table: StudyBotMain
- Record exists after upload

### IAM role
![IAM role](screenshots/day1/iam_role.png)
- No AdministratorAccess
- No wildcard Action: *

---

## 6. AI Evidence

### Sample input file
- File: docs/sample_input/lecture.pdf
- Type: PDF with table/figure/caption

### Summary output
![Summary](screenshots/day2/summary_output.png)

### Q&A with citation
![Q&A citation](screenshots/day2/qa_citations.png)
- Citation format: [filename - chunk_id]

### Quiz output
![Quiz](screenshots/day2/quiz_output.png)
- 5 MCQ questions generated

### Retrieval evidence
- Chunking strategy: [điền — semantic vs fixed, chunk size]
- Failure mode found: [điền — tên query bị sai, đã fix thế nào]

---

## 7. Security and IAM

### IAM least-privilege policy
Lambda execution role permissions:
- s3:PutObject, s3:GetObject, s3:ListBucket
- dynamodb:PutItem, dynamodb:GetItem, dynamodb:Query, dynamodb:UpdateItem
- bedrock:InvokeModel
- bedrock-agent-runtime:RetrieveAndGenerate
- logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents
- cloudwatch:PutMetricData

### S3 security
- Block Public Access: ON
- Versioning: ON
- Default encryption: ON

### Network isolation
- Lambda in private subnet
- DynamoDB via Gateway Endpoint (free, no NAT Gateway)
- Bedrock via Interface Endpoint

---

## 8. Observability

### CloudWatch dashboard
![Dashboard](screenshots/day2/cloudwatch_dashboard.png)
- Widgets: Lambda invocations, Lambda errors, StudyBot/DocumentsUploaded

### Custom metric
- Metric name: StudyBot/DocumentsUploaded
- Published via PutMetricData after each successful upload

### Alarm
![Alarm](screenshots/day2/cloudwatch_alarm.png)
- Alarm on Lambda Errors metric
- State: OK (not INSUFFICIENT_DATA)

### Log Insights query
- Query: filter ERROR, Exception, AccessDenied — sort desc — limit 20
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
1. OpenSearch Serverless — ~$27.65 — vector store cho Bedrock KB
2. Bedrock (Haiku + Titan Embeddings) — ~$1-2 — AI calls
3. VPC Interface Endpoint (Bedrock) — ~$0.62 — private connectivity


### Cost control notes
- Used Claude Haiku not Sonnet in dev
- No NAT Gateway — used VPC endpoints instead
- Single-AZ, no Multi-AZ
- Deleted unused resources daily
- Total spend: $[điền]

---

## 10. Lessons Learned

[Điền cuối Day 2 — khoảng 200 chữ]

Gợi ý:
- Điều gì hoạt động tốt?
- Điều gì thất bại hoặc mất nhiều thời gian hơn dự kiến?
- Nếu làm lại, sẽ thay đổi gì?
- Nếu Khanmigo engineer review architecture này, họ sẽ nói gì?

---

## 11. Teardown Plan

Delete resources theo thứ tự:
1. CloudFront — disable trước, rồi delete
2. API Gateway — delete
3. Lambda functions — delete
4. Bedrock KB / OpenSearch Serverless — delete nếu dùng
5. DynamoDB table — delete
6. S3 buckets — empty trước, rồi delete
7. CloudWatch dashboards/alarms/log groups — delete
8. IAM roles/policies — delete
9. VPC endpoints / networking — delete cuối cùng

Teardown confirmation file: docs/teardown_confirmation.md
- List resource đã xóa
- Final Cost Explorer screenshot (chụp ngày 2/6)
- Tên người verify teardown
