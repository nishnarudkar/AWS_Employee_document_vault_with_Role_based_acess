# AWS Employee Document Vault with Role-Based Access Control

A secure, serverless enterprise document management system built on AWS. The system provides authenticated, fine-grained role-based access control (RBAC) to confidential employee records — such as payslips, offer letters, and appraisal documents — through a RESTful API powered by AWS Lambda, Amazon S3, Amazon DynamoDB, and Amazon Cognito.

---

## Table of Contents

- [Overview](#overview)
- [Live Demo](#live-demo)
- [Architecture](#architecture)
- [AWS Service Architecture](#aws-service-architecture)
- [Access Control & Authorization Model](#access-control--authorization-model)
- [Document Storage & Database Schema](#document-storage--database-schema)
- [API Reference](#api-reference)
- [Monitoring, Observability & Alerting](#monitoring-observability--alerting)
- [Performance & Load Testing Results](#performance--load-testing-results)
- [Project Artifacts & Documentation](#project-artifacts--documentation)
- [Repository Structure](#repository-structure)
- [CI/CD Deployment Pipeline](#cicd-deployment-pipeline)
- [Automated Testing](#automated-testing)
- [Security Hardening & Best Practices](#security-hardening--best-practices)
- [Future Enhancements](#future-enhancements)
- [License](#license)

---

## Overview

The AWS Employee Document Vault centralizes corporate HR records while enforcing strict access boundaries based on authenticated user identity and organizational role.

### Core Features

- **Fine-Grained Role-Based Authorization:** Dynamically restricts document access based on user roles embedded in Cognito JWT tokens.
- **Serverless Architecture:** Completely pay-per-use, highly scalable backend utilizing AWS serverless primitives.
- **Audit Traceability:** Comprehensive tracking of all write, download, and soft-delete actions in DynamoDB.
- **Automated Deployment:** CI/CD pipeline using GitHub Actions with keyless AWS authentication via OpenID Connect (OIDC).
- **Enterprise Observability:** Distributed request tracing via AWS X-Ray and CloudWatch latency/error rate alarms with SNS notifications.

---

## Live Demo

- **Frontend Endpoint:** [http://vault-hr-frontend-nikitha-2026.s3-website-us-east-1.amazonaws.com](http://vault-hr-frontend-nikitha-2026.s3-website-us-east-1.amazonaws.com)

### Pre-configured Demo Credentials

| Role | Username | Password | Scope of Access |
|---|---|---|---|
| Employee | `emp001` | `Employee1@123` | Access restricted strictly to own personal documents |
| Manager | `mgr001` | `Manager@123` | Access restricted to documents of direct reports |
| HR Admin | `hr001` | `Hr001@123` | Full access across all organization document records |

*Note: Demo credentials are configured for evaluation in a non-production test environment.*

---

## Architecture

![Architecture Diagram](architecture%20diagram/WhatsApp%20Image%202026-09-16%20at%2000.51.29.jpeg)

---

## AWS Service Architecture

| AWS Service | Operational Purpose |
|---|---|
| **Amazon Cognito** | User directory, authentication, and JWT token issuing with custom groups |
| **Amazon API Gateway** | Regional REST API endpoint management and Cognito Authorizer integration |
| **AWS Lambda** | Serverless backend business logic and resource-level authorization |
| **Amazon S3** | Encrypted document file storage with enabled object versioning |
| **Amazon DynamoDB** | Single-digit millisecond latency storage for document metadata and audit logs |
| **AWS IAM** | Granular service-to-service execution roles following least-privilege principles |
| **Amazon CloudWatch** | Real-time metrics, custom dashboards, latency alarms, and SNS notifications |
| **AWS X-Ray** | Distributed request tracing and end-to-end trace maps |
| **Amazon SNS** | Push notification service for automated threshold alert delivery |

---

## Access Control & Authorization Model

Authorization is enforced deterministically across multiple security boundaries:

```
Client Request (Bearer JWT)
       │
       ▼
Amazon API Gateway (Cognito Authorizer Validation)
       │
       ▼
AWS Lambda Execution (Role Claim Extraction & Resource Check)
       │
       ▼
AWS IAM Role Scope (Least-Privilege Resource Access)
       │
       ▼
Target Resource (Amazon S3 / DynamoDB)
```

### Role Matrix

- **Employee (`emp001`):** Permitted to read/download/list files where `employee_id == requester_id`.
- **Manager (`mgr001`):** Permitted to read/download/list files where `manager_id == requester_id`.
- **HR Admin (`hr001`):** Unrestricted read, upload, soft-delete, and audit inspection permissions across all organization documents.

---

## Document Storage & Database Schema

### S3 Storage Layout

Documents are stored in Amazon S3 adhering to a structured partition path:

```
documents/{employee_id}/{document_type}/{filename}
```

*Example S3 Object Key:*
`documents/emp001/PaySlip/EMP001_August_2026_Payslip.pdf`

### DynamoDB Document Metadata Schema

| Attribute | Type | Description |
|---|---|---|
| `document_id` | String (PK) | Unique document identifier (e.g. `DOC3F9A1C...`) |
| `employee_id` | String | Target employee username |
| `manager_id` | String | Assigned manager username |
| `uploaded_by` | String | Username of the uploader |
| `document_type` | String | Category (`PaySlip`, `OfferLetter`, `Appraisal`) |
| `file_name` | String | Original document file name |
| `s3_key` | String | Full S3 object path |
| `upload_timestamp` | String | ISO 8601 UTC timestamp |
| `tags` | List | Metadata keywords |
| `deleted` | Boolean | Soft-deletion status flag |

---

## API Reference

**Base Endpoint:** `https://c9d8wcytpj.execute-api.us-east-1.amazonaws.com/prod`

All HTTP requests must include a valid Bearer token in the `Authorization` header:
`Authorization: Bearer <COGNITO_JWT_TOKEN>`

| Method | Endpoint | Authorization | Description |
|---|---|---|---|
| `POST` | `/upload` | HR_Admin | Generates pre-signed S3 upload URL and creates document record |
| `GET` | `/files` | Authenticated Users | Retrieves list of documents filtered by user role scope |
| `GET` | `/download/{doc_id}` | Authorized Users | Generates pre-signed S3 download URL for requested file |
| `DELETE` | `/files/{doc_id}` | HR_Admin | Executes soft-delete on target document record |

---

## Monitoring, Observability & Alerting

The application leverages Amazon CloudWatch and AWS X-Ray for enterprise-grade monitoring.

### CloudWatch Operational Dashboard
![CloudWatch Dashboard](Screenshots/Employee%20Document%20Vault%20-%20CloudWatch%20Dashboard.png)

### End-to-End AWS X-Ray Trace Map
![X-Ray Trace Map](Screenshots/Employee%20Document%20Vault%20-%20Trace%20map.png)

### Automated Alarms & SNS Alerting
- **Latency Alarm:** Triggers when API latency breaches operational SLAs.
  ![Latency Alarm](Screenshots/Employee%20Document%20Vault%20-%20Latency%20Alarm.png)
- **Error Rate Alarm:** Triggers on HTTP 5xx or execution failure spikes.
  ![Error Rate Alarm](Screenshots/Employee%20Document%20Vault%20-%20Error%20rate%20Alarm.png)
- **SNS Topic Subscriptions:** Sends real-time notifications to administration teams upon alarm state changes.
  ![SNS Notifications](Screenshots/Employee%20Document%20Vault%20-%20SNS.png)

---

## Performance & Load Testing Results

The production system was evaluated under load using **Artillery 2.0.33** executing against the `/files` endpoint in `us-east-1`.

### Artillery Test Summary

| Metric | Measured Value |
|---|---|
| Total HTTP Requests | 600 |
| Successful Responses (HTTP 200) | 600 (100% Success) |
| Failed Requests | 0 |
| Request Throughput | 5 requests/sec |
| Virtual Users | 10 concurrent users |
| Minimum Latency | 236 ms |
| Mean Latency | 440.2 ms |
| Median Latency (P50) | 424.2 ms |
| 95th Percentile Latency (P95) | 685.5 ms |
| 99th Percentile Latency (P99) | 1,224.4 ms |
| Maximum Latency | 1,396 ms |

### CloudWatch Lambda Metrics Verification (`listDocuments`)

| Metric | Result |
|---|---|
| Invocations | 605 |
| Executions Errors | 0 |
| Success Rate | 100% |
| Throttles | 0 |
| Max Concurrent Executions | 10 |
| Average Execution Duration | 215 ms |

---

## Project Artifacts & Documentation

The repository contains comprehensive project evaluation artifacts and documentation files:

1. **[Project Report](Employee_Document_Vault_Project_Report.docx):** Full architectural design document and implementation report.
2. **[Security Hardening Case Study](Employee_Document_Vault_Security_Hardening_Case_Study_Mentor_Ready_v2.pdf):** In-depth security review, threat modeling, and mitigation strategy.
3. **[Cost Estimation Model](Employee_Document_Vault_Cost_Estimation.xlsx):** Detailed AWS pricing breakdown and monthly cost projection under varying load tiers.
4. **[Artillery Load Test Report](Artillery_LoadTest_Report.pdf):** Detailed load test performance benchmarks and latency breakdown.

---

## Repository Structure

```
Employee_Document_Vault_with_Role_based_Access/
├── .github/
│   └── workflows/
│       └── deploy.yml                                # GitHub Actions CI/CD pipeline
├── architecture diagram/
│   └── WhatsApp Image 2026-09-16 at 00.51.29.jpeg    # Architecture diagram image
├── lambda/
│   ├── deleteDocument/
│   │   └── lambda_function.py                        # Soft-delete Lambda handler
│   ├── downloadDocument/
│   │   └── lambda_function.py                        # Pre-signed download URL generator
│   ├── listDocuments/
│   │   └── lambda_function.py                        # Role-filtered document lister
│   └── uploadDocument/
│       └── lambda_function.py                        # Pre-signed upload URL generator
├── Screenshots/                                      # Monitoring & alarm screenshots
├── tests/
│   └── test_basic.py                                 # Automated Python test suite
├── .gitignore                                        # Excluded files & environment rules
├── Artillery_LoadTest_Report.pdf                     # Artillery performance test report
├── Employee_Document_Vault_Cost_Estimation.xlsx      # AWS cost estimation spreadsheet
├── Employee_Document_Vault_Project_Report.docx       # Project documentation report
├── Employee_Document_Vault_Security_Hardening...pdf  # Security case study
└── Readme.md                                         # Project documentation
```

---

## CI/CD Deployment Pipeline

The project utilizes GitHub Actions (`.github/workflows/deploy.yml`) for continuous integration and automated deployment to AWS.

### Pipeline Workflow Steps

1. **Code Checkout & Setup:** Checks out repository code and initializes Python 3.12 environment.
2. **Automated Testing:** Runs `pytest` suite across all Lambda function modules.
3. **Keyless AWS Authentication:** Authenticates to AWS via **GitHub OIDC** assuming a dedicated IAM role (eliminating long-lived secret keys).
4. **Packaging & Deployment:** Compresses Lambda source code and deploys updates using `aws lambda update-function-code`.

---

## Automated Testing

Automated sanity and syntax verification tests are executed using `pytest`.

### Running Tests Locally

```bash
pip install pytest
pytest -q
```

The test suite validates:
- Presence of required Lambda directory structures.
- Syntactic correctness of Python handlers.
- Export of valid `lambda_handler` entry points.

---

## Security Hardening & Best Practices

- **Zero Trust Authentication:** API Gateway enforces Cognito authorizer validation before passing requests to compute layers.
- **Data Encryption:** Server-Side Encryption enabled on S3 buckets (`SSE-S3`) and DynamoDB tables.
- **Short-Lived Pre-Signed URLs:** S3 object access occurs strictly via time-limited pre-signed URLs (15-minute expiration).
- **Soft Deletion Policy:** Records are marked `deleted = true` in DynamoDB, maintaining object history without immediate permanent data loss.
- **OIDC Deployment Security:** Deployment pipeline relies on AWS IAM OIDC federation tied specifically to the repository `main` branch.

---

## Future Enhancements

- **Direct S3 Multipart Pre-Signed Uploads:** Support client-side direct S3 uploads for large file payloads.
- **OpenSearch Integration:** Enable full-text search capability across document metadata and file content.
- **Automated Infrastructure as Code (IaC):** Provision resources using Terraform or AWS CDK.
- **Advanced Audit Retention:** Export DynamoDB audit trails to Amazon S3 Glacier for long-term compliance storage.

---

## License

This project was developed as an enterprise serverless document management system reference implementation on AWS.
