# AWS Infrastructure Monitoring & RCA Project Plan

## Project Overview
Build a comprehensive monitoring solution that collects AWS infrastructure metrics, custom application metrics, and business metrics (Upstream Market Data Availability, Asset Classes Batch Processing, Critical Sub-stages), stores them in Snowflake, and provides natural language querying with AI-powered root cause analysis.

## Architecture Overview
```
AWS Services → Lambda Collectors → S3 (Raw) → Snowflake → OpenAI LLM → Dashboard/Alerts
```

---

## JIRA Project Structure

### Project: **CMS** (Central Metric Store)

---

## EPIC 1: Infrastructure Foundation Setup
**Epic Key**: CMS-1  

### Stories:

#### CMS-2: S3 Bucket Configuration and Lifecycle Management

**Description**: Set up S3 infrastructure for metrics storage with proper lifecycle policies and security configurations.

**Acceptance Criteria**:
- [ ] Create dedicated S3 buckets for raw metrics, processed metrics, and archival
- [ ] Configure lifecycle policies (IA after 30 days, Glacier after 90 days)
- [ ] Set up bucket versioning and KMS encryption
- [ ] Create folder structure: 

---

#### CMS-5: CloudWatch Monitoring and Alerting Foundation
**Story Points**: 8  
**Assignee**: DevOps Engineer  
**Sprint**: 2  

**Description**: Establish comprehensive monitoring for the monitoring system itself.

**Acceptance Criteria**:
- [ ] Set up CloudWatch log groups for all Lambda functions
- [ ] Configure CloudWatch alarms for Lambda failures, timeouts, and memory usage
- [ ] Create CloudWatch dashboard for system health
- [ ] Set up SNS topics for alert notifications
- [ ] Configure log retention policies

---

#### CMS-8: Snowflake Environment Setup

**Description**: Set up complete Snowflake environment with optimized configuration.

**Acceptance Criteria**:
- [ ] Configure Snowflake warehouses (XS for regular queries, M for analytics)
- [ ] Set up databases, schemas, and table structures
- [ ] Configure user roles and permissions
- [ ] Create external stages pointing to S3 buckets
- [ ] Test data loading from S3

---

## EPIC 2: Native AWS Metrics Collection

### Stories:

#### CMS-11: FGW RDS Metrics Collection Lambda

**Description**: Implement comprehensive RDS metrics collection with OTEL formatting.

**Acceptance Criteria**:
- [ ] Collect CPU utilization, memory usage, storage metrics
- [ ] Monitor connection count, read/write IOPS, latency
- [ ] Convert all metrics to OTEL format

---

#### CMS-12: SQS Metrics Collection Lambda
**Story Points**: 13  
**Assignee**: Backend Developer  
**Sprint**: 4  

**Description**: Implement SQS metrics collection for queue monitoring and alerting.

**Acceptance Criteria**:
- [ ] Collect queue depth and message age metrics
- [ ] Monitor receive/send message counts and rates
- [ ] Monitor visibility timeout and message retention
---

#### CMS-13: Lambda Function Metrics Collection

**Description**: Monitor Lambda function performance and health metrics.

**Acceptance Criteria**:
- [ ] Collect function duration and memory usage
- [ ] Monitor error rates, throttles, and timeouts
- [ ] Track concurrent executions and provisioned concurrency

#### CMS-14: EKS Function Metrics Collection

**Description**: Monitor Lambda function performance and health metrics.

**Acceptance Criteria**:
- [ ] Collect function duration and memory usage
- [ ] Monitor error rates, throttles, and timeouts
- [ ] Track concurrent executions and provisioned concurrency

---


## EPIC 3: Business Metrics SDK and Integration

### Stories:

#### CMS-20: Upstream Market Data Availability Metrics

**Description**: Implement monitoring for upstream market data feeds availability and quality.

**Acceptance Criteria**:
- [ ] Monitor real-time data feed connectivity status
- [ ] Monitor data quality metrics (missing fields, invalid values)
- [ ] Generate alerts for data feed interruptions


---

#### CMS-21: Asset Classes Batch Processing Metrics

**Description**: Comprehensive monitoring of batch processing jobs for different asset classes.

**Acceptance Criteria**:
- [ ] Track batch job start and end times for each asset class
- [ ] Monitor processing times and performance trends
- [ ] Calculate processing rates (records per minute)
- [ ] Track batch job success/failure rates
- [ ] Monitor data volume processed per batch
- [ ] Generate SLA compliance reports


#### CMS-22: Critical Sub-stages Processing Metrics
**Story Points**: 21  
**Assignee**: Application Developer  
**Sprint**: 8  

**Description**: Monitor critical sub-stages within batch processing workflows.

**Acceptance Criteria**:
- [ ] Track start and end times for each critical sub-stage
- [ ] Monitor processing times for data validation, transformation, and loading stages
- [ ] Track error rates and retry attempts for each sub-stage
- [ ] Create dependency mapping between sub-stages


---

#### CMS-23: Business Metrics SDK Core Library

**Description**: Create lightweight SDK for application teams to publish business metrics.

**Acceptance Criteria**:
- [ ] Support for Python, Java, and Node.js
- [ ] Built-in retry logic and error handling
- [ ] Configuration management for different environments
- [ ] Performance impact monitoring


---


## EPIC 4: Data Pipeline and Snowflake Schema

### Stories:

#### CMS-27: Snowflake Schema Design for Financial Metrics

**Description**: Design optimized Snowflake schema for financial and infrastructure metrics.


---


---

## EPIC 5: OpenAI LLM Integration for RCA

### Stories:

#### CMS-33: Natural Language Query Interface
**Story Points**: 21  
**Assignee**: AI/ML Engineer  
**Sprint**: 13-14  

**Description**: Create sophisticated natural language processing for financial metrics queries.

**Query Examples**:
- "Show me all derivatives batch jobs that failed in the last 2 hours"
- "Which upstream data feeds was delayed today?"
- "Are there any critical sub-stages taking longer than usual today?"


---

#### CMS-34: Financial Domain Context Engine

**Description**: Build domain-specific context understanding for financial operations.

**Domain Knowledge Areas**:

**Technical Tasks**:
- Build financial domain knowledge base
- Create asset class and workflow ontologies
- Implement business context enrichment
- Set up domain-specific validation rules

---

#### CMS-35: Root Cause Analysis Engine



**RCA Scenarios**:

1. **Performance Degradations**
   - Database connection pool exhaustion
   - Memory leaks in long-running processes
   - Network latency to external data providers

2. **SLA Violations**
   - Delayed market data affecting batch start times
   - Unexpected data volume spikes
   - Infrastructure capacity constraints


---


## EPIC 6: Dashboard and User Interface

### Stories:

#### CMS-38: Financial Operations Dashboard

**Description**: Comprehensive dashboard for financial operations monitoring.

**Dashboard Components**:
1. **Market Data Feed Status**
   - Real-time feed availability grid
   - Data quality indicators
   - Latency heat maps

2. **Batch Processing Overview**
   - Active batch jobs timeline
   - Processing duration trends
   - Success/failure rates by asset class
   - SLA compliance tracking

3. **Critical Sub-stages Monitor**
   - Stage-by-stage processing progress
   - Bottleneck identification
   - Resource utilization tracking
   - Performance trend analysis
