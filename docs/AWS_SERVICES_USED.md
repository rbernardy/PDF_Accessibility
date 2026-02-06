# AWS Services Used in PDF Accessibility Solution

## Complete List of AWS Services

This document lists all AWS services used by the PDF Accessibility solution, organized by category.

---

## Core Compute Services

### 1. **AWS Lambda**
- **Purpose**: Serverless compute for PDF processing functions
- **Functions**:
  - `split_pdf` - Splits PDFs into chunks for parallel processing
  - `accessibility_checker_before_remidiation` - Runs accessibility checks before remediation
  - `accessability_checker_after_remidiation` - Runs accessibility checks after remediation
  - `add_title` - Generates and adds titles to PDFs using AI
  - `java_lambda` (PDF Merger) - Merges processed PDF chunks back together
  - `pdf2html_lambda` - Converts PDFs to accessible HTML (pdf2html solution)
- **Used in**: Both PDF-to-PDF and PDF-to-HTML solutions

### 2. **Amazon ECS (Elastic Container Service)**
- **Purpose**: Container orchestration for long-running PDF processing tasks
- **Task Definitions**:
  - **Python Container** (`docker_autotag`) - Adobe PDF Services autotagging and extraction
  - **JavaScript Container** (`javascript_docker`) - AI-powered alt-text generation for images
- **Configuration**: 
  - Fargate launch type (serverless containers)
  - 2 vCPU, 4GB memory per task
  - Private subnet deployment with NAT Gateway
- **Used in**: PDF-to-PDF solution only

### 3. **Amazon ECR (Elastic Container Registry)**
- **Purpose**: Stores Docker images for ECS tasks
- **Images**:
  - Python autotag container image
  - JavaScript alt-text container image
- **Used in**: PDF-to-PDF solution only

---

## Orchestration & Workflow

### 4. **AWS Step Functions**
- **Purpose**: Orchestrates the multi-step PDF processing workflow
- **Workflow Steps**:
  1. Accessibility check (before)
  2. Map state for parallel chunk processing
  3. ECS Task 1 (Adobe autotagging)
  4. ECS Task 2 (AI alt-text generation)
  5. Java Lambda (merge chunks)
  6. Add Title Lambda (AI title generation)
  7. Accessibility check (after)
- **Features**:
  - Parallel processing with Map state
  - Error handling and retries
  - State machine logging to CloudWatch
- **Used in**: PDF-to-PDF solution only

---

## Storage Services

### 5. **Amazon S3 (Simple Storage Service)**
- **Purpose**: Primary storage for PDFs and processing artifacts
- **Bucket Structure**:
  - `pdf/` - Input PDFs (with optional subfolder structure)
  - `temp/` - Intermediate processing files
    - `temp/{folder_path}/{filename}/` - Chunk files
    - `temp/{folder_path}/{filename}/output_autotag/` - Autotagged files
    - `temp/{folder_path}/{filename}/accessability-report/` - Accessibility reports
  - `result/` - Final compliant PDFs (preserves folder structure)
  - `output/` - HTML output (pdf2html solution)
- **Features**:
  - Server-side encryption (SSE-S3)
  - SSL enforcement
  - CORS configuration for web access
  - Event notifications trigger Lambda functions
- **Used in**: Both solutions

---

## AI/ML Services

### 6. **Amazon Bedrock**
- **Purpose**: Generative AI for content generation
- **Models Used**:
  - **Claude 3.5 Sonnet** (`us.anthropic.claude-3-5-sonnet-20241022-v2:0`) - Image alt-text generation
  - **Claude 3 Haiku** (`us.anthropic.claude-3-haiku-20240307-v1:0`) - Link text generation
  - **Nova Pro** (`us.amazon.nova-pro-v1:0`) - PDF title generation
- **Use Cases**:
  - Generate descriptive alt-text for images
  - Generate accessible link text
  - Generate document titles from content
- **Used in**: PDF-to-PDF solution

### 7. **Amazon Bedrock Data Automation (BDA)**
- **Purpose**: Automated PDF to HTML conversion with accessibility features
- **Features**:
  - Extracts text, images, tables from PDFs
  - Generates semantic HTML structure
  - Preserves document layout
- **Used in**: PDF-to-HTML solution only

### 8. **Amazon Comprehend**
- **Purpose**: Natural language processing for language detection
- **Use Case**: Detects dominant language in PDF content for metadata
- **API**: `detect_dominant_language`
- **Used in**: PDF-to-PDF solution (docker_autotag)

---

## Security & Secrets Management

### 9. **AWS Secrets Manager**
- **Purpose**: Securely stores API credentials and secrets
- **Secrets Stored**:
  - `/myapp/client_credentials` - Adobe PDF Services API credentials
    - `PDF_SERVICES_CLIENT_ID`
    - `PDF_SERVICES_CLIENT_SECRET`
  - `/myapp/db_credentials` - Database credentials (if used)
- **Access**: Lambda functions and ECS tasks retrieve secrets at runtime
- **Used in**: PDF-to-PDF solution

### 10. **AWS IAM (Identity and Access Management)**
- **Purpose**: Access control and permissions
- **Roles Created**:
  - ECS Task Execution Role
  - ECS Task Role
  - Lambda Execution Roles (per function)
  - Step Functions Execution Role
  - CodeBuild Service Role
- **Policies**:
  - S3 read/write access
  - Bedrock invoke model permissions
  - Secrets Manager read access
  - CloudWatch Logs write access
  - Step Functions execution permissions
- **Used in**: Both solutions

---

## Networking

### 11. **Amazon VPC (Virtual Private Cloud)**
- **Purpose**: Network isolation for ECS tasks
- **Configuration**:
  - 2 Availability Zones
  - Public subnets (for NAT Gateway)
  - Private subnets (for ECS tasks)
  - 1 NAT Gateway for outbound internet access
- **Security Groups**: Control inbound/outbound traffic for ECS tasks
- **Used in**: PDF-to-PDF solution (for ECS)

---

## Monitoring & Logging

### 12. **Amazon CloudWatch**
- **Purpose**: Monitoring, logging, and observability
- **Components**:
  - **CloudWatch Logs**: 
    - Lambda function logs
    - ECS container logs
    - Step Functions execution logs
  - **CloudWatch Metrics**: Custom metrics for processing status
  - **CloudWatch Dashboards**: 
    - File processing status
    - Lambda execution logs
    - ECS task logs
    - Step Functions execution logs
- **Log Groups**:
  - `/aws/lambda/split_pdf`
  - `/aws/lambda/java_lambda`
  - `/aws/lambda/add_title`
  - `/aws/lambda/a11y_precheck`
  - `/aws/lambda/a11y_postcheck`
  - `/ecs/MyFirstTaskDef/PythonContainerLogGroup`
  - `/ecs/MySecondTaskDef/JavaScriptContainerLogGroup`
  - `/aws/vendedlogs/states/PDFAccessibility-StateMachine`
- **Used in**: Both solutions

---

## CI/CD & Deployment

### 13. **AWS CodeBuild**
- **Purpose**: Builds and deploys the CDK application
- **Build Specifications**:
  - `buildspec-unified.yml` - Unified build for both solutions
- **Build Environment**:
  - Amazon Linux 2 or Amazon Linux x86_64
  - Docker support (privileged mode)
  - Compute: BUILD_GENERAL1_SMALL or BUILD_GENERAL1_LARGE
- **Process**:
  1. Clones GitHub repository
  2. Installs dependencies (Python, Node.js, CDK)
  3. Builds Docker images
  4. Synthesizes CDK stack
  5. Deploys to AWS
- **Used in**: Both solutions (via deploy.sh)

### 14. **AWS CloudFormation**
- **Purpose**: Infrastructure as Code deployment
- **How Used**: CDK synthesizes to CloudFormation templates
- **Stack Name**: `PDFAccessibility`
- **Resources Managed**: All AWS resources listed in this document
- **Used in**: Both solutions

---

## Database (Optional - PDF-to-HTML)

### 15. **Amazon DynamoDB**
- **Purpose**: Job status tracking for batch processing
- **Table**: Job status table
- **Operations**:
  - Create job records
  - Update job status
  - Query job status
- **Used in**: PDF-to-HTML solution (batch processing)

### 16. **Amazon SQS (Simple Queue Service)**
- **Purpose**: Message queue for batch job processing
- **Use Case**: Queues PDF processing jobs for asynchronous processing
- **Used in**: PDF-to-HTML solution (batch processing)

---

## Identity & Authentication (Frontend Only)

### 17. **Amazon Cognito**
- **Purpose**: User authentication for frontend UI
- **Features**:
  - User pools for authentication
  - User sign-up and sign-in
  - Password management
- **Used in**: Frontend UI only (not in backend solutions)

---

## Third-Party Services (Not AWS)

### Adobe PDF Services API
- **Purpose**: Advanced PDF operations
- **Operations**:
  - PDF autotagging for accessibility
  - Extract structured data (text, tables, figures)
  - Generate accessibility reports
- **Integration**: Called from ECS Python container and Lambda functions
- **Credentials**: Stored in AWS Secrets Manager

---

## Service Usage by Solution Type

### PDF-to-PDF Solution Uses:
1. ✅ Lambda (5 functions)
2. ✅ ECS (2 containers)
3. ✅ ECR (2 images)
4. ✅ Step Functions
5. ✅ S3
6. ✅ Bedrock (3 models)
7. ✅ Comprehend
8. ✅ Secrets Manager
9. ✅ IAM
10. ✅ VPC
11. ✅ CloudWatch (Logs, Metrics, Dashboards)
12. ✅ CodeBuild
13. ✅ CloudFormation

**Total: 13 AWS Services**

### PDF-to-HTML Solution Uses:
1. ✅ Lambda (1 function)
2. ✅ S3
3. ✅ Bedrock Data Automation
4. ✅ IAM
5. ✅ CloudWatch (Logs)
6. ✅ CodeBuild
7. ✅ CloudFormation
8. ✅ DynamoDB (optional, for batch)
9. ✅ SQS (optional, for batch)

**Total: 7-9 AWS Services**

### Frontend UI Uses:
1. ✅ S3 (static hosting)
2. ✅ Cognito
3. ✅ CloudFront (CDN)
4. ✅ API Gateway (connects to backend)
5. ✅ IAM

**Total: 5 AWS Services**

---

## Cost Considerations

### High-Cost Services:
1. **ECS Fargate** - Charged per vCPU-hour and GB-hour
2. **NAT Gateway** - Charged per hour + data processing
3. **Bedrock** - Charged per 1000 input/output tokens
4. **Step Functions** - Charged per state transition

### Low-Cost Services:
1. **Lambda** - Free tier: 1M requests/month, 400,000 GB-seconds
2. **S3** - Free tier: 5GB storage, 20,000 GET requests
3. **CloudWatch Logs** - Free tier: 5GB ingestion, 5GB storage

### Cost Optimization Tips:
- Use S3 lifecycle policies to delete temp files
- Set CloudWatch log retention periods
- Use Lambda instead of ECS where possible
- Monitor Bedrock token usage
- Delete old ECS task definitions and ECR images

---

## Regional Considerations

Most services are deployed in a single AWS region (configurable):
- Default: `us-east-1`
- Can be changed via CDK context or environment variables

**Services with Regional Availability Considerations:**
- **Bedrock**: Not available in all regions (check AWS documentation)
- **Bedrock Data Automation**: Limited regional availability
- **ECS Fargate**: Available in most regions
- **Step Functions**: Available in all commercial regions

---

## Service Limits to Monitor

1. **Lambda**:
   - Concurrent executions: 1,000 (default)
   - Function timeout: 15 minutes max
   - Deployment package size: 250 MB

2. **ECS**:
   - Tasks per service: 1,000
   - Task definitions: 1,000 revisions per family

3. **Step Functions**:
   - Execution history: 90 days
   - Execution time: 1 year max

4. **S3**:
   - Bucket limit: 100 per account (soft limit)
   - Object size: 5 TB max

5. **Bedrock**:
   - Requests per minute: Varies by model
   - Token limits: Varies by model

---

## Summary

**Total AWS Services Used: 17**

The solution is a comprehensive, production-ready system leveraging AWS's serverless and container services for scalable PDF accessibility remediation. The architecture is designed for:
- ✅ High availability (multi-AZ)
- ✅ Scalability (serverless + auto-scaling)
- ✅ Security (encryption, IAM, Secrets Manager)
- ✅ Observability (CloudWatch monitoring)
- ✅ Cost optimization (pay-per-use model)
