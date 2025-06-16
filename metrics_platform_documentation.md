# Central Metric Collection Platform - Technical Documentation

## Table of Contents
1. [Types of Metrics](#types-of-metrics)
2. [Storage Architecture on S3](#storage-architecture-on-s3)
3. [Storage Format for Infra and Business Metrics](#storage-format-for-infra-and-business-metrics)
4. [Snowflake Table Creation](#snowflake-table-creation)
5. [Sample RCA Query](#sample-rca-query)
6. [Sample Query Results & Analysis](#sample-query-results--analysis)

---

## Types of Metrics

### 1. Infrastructure Metrics
**Collection Method**: Automated pull via Lambda every 15 minutes  
**Purpose**: Monitor AWS service health and performance  

**Services Covered**:
- **EKS**: Pod crashes, CPU utilization, memory usage, storage utilization
- **RDS**: Select latency, CPU utilization, database connections, read/write latency
- **Lambda**: Total executions, error rate, throttle count, duration
- **SQS**: Message age, visible messages, in-flight messages
- **DynamoDB**: Job requests, cluster status, EMR cluster health
- **S3**: Read/write I/O operations, small file count
- **EMR**: Step submission rates, Graviton node count

### 2. Business Metrics

#### 2.1 Markers
**Collection Method**: Real-time push on data arrival  
**Purpose**: Track upstream data dependency SLAs  

**Examples**:
- `GLOBAL-CCCCollateral-EOD_Close`
- `GLOBAL-MarketableCollateral-EOD_Close`
- `GLOBAL-OTCRegDerivatives-EOD_Close`
- `GLOBAL-BRKRACC-EOD_Close`
- `GLOBAL-ExchangeTradedDerivatives-EOD_Close`
- `GLOBAL-Security-EOD_Close`
- `GLOBAL-SFT-EOD_Close`
- `GLOBAL-FnOCollateral-EOD_Close`

#### 2.2 Milestones
**Collection Method**: Push on process completion  
**Purpose**: Monitor critical daily batch processing SLAs  

**Examples**:
- Deposit processing
- DERIVATIVES processing
- COLLATERAL processing
- SECURITIES processing
- Loan processing
- SNU processing
- UPC processing
- STRESS_SNAPSHOT
- FINAL_STRESS_END
- 6G submission
- 6G table finish
- SLS_LOCK

#### 2.3 SLA Metrics
**Collection Method**: Continuous push from Airflow DAGs  
**Purpose**: Track workflow execution and SLA compliance  

---

## Storage Architecture on S3

### Directory Structure
```
s3://metrics-bucket/
├── infra/
│   ├── year=2025/month=06/day=15/service=eks/
│   │   └── infra_metrics_2025-06-15.json
│   ├── year=2025/month=06/day=15/service=lambda/
│   │   └── infra_metrics_2025-06-15.json
│   ├── year=2025/month=06/day=15/service=dynamodb/
│   │   └── infra_metrics_2025-06-15.json
│   ├── year=2025/month=06/day=15/service=sqs/
│   │   └── infra_metrics_2025-06-15.json
│   ├── year=2025/month=06/day=15/service=rds/
│   │   └── infra_metrics_2025-06-15.json
│   ├── year=2025/month=06/day=15/service=s3/
│   │   └── infra_metrics_2025-06-15.json
│   └── year=2025/month=06/day=15/service=emr/
│       └── infra_metrics_2025-06-15.json
├── business/
│   ├── year=2025/month=06/day=15/domain=markers/
│   │   └── business_metrics_2025-06-15.json
│   ├── year=2025/month=06/day=15/domain=milestones/
│   │   └── business_metrics_2025-06-15.json
│   └── year=2025/month=06/day=15/domain=sla/
│       └── business_metrics_2025-06-15.json
└── archive/
    ├── infra/
    │   └── year=2024/month=12/day=31/
    └── business/
        └── year=2024/month=12/day=31/
```

### Design Principles
- **Partitioned by Date**: Enables efficient time-based querying
- **Partitioned by Service/Domain**: Allows targeted data retrieval
- **Single Daily Files**: One JSON file per service/domain per day
- **Append-Only Updates**: Files updated as new metrics arrive
- **Archive Strategy**: Historical data moved to archive for cost optimization

---

## Storage Format for Infra and Business Metrics

### Common JSON Keys (All Metric Types)

| Key | Description | Example | Purpose |
|-----|-------------|---------|---------|
| `schema_version` | Format version for evolution | `"1.0"` | Backward compatibility |
| `date` | Collection date | `"2025-06-15"` | Partitioning key |
| `environment` | Environment identifier | `"prod"` | Multi-env support |
| `readings` | Array of metric collections | `[{...}]` | Time-series data |

### Infrastructure Metrics Schema

```json
{
  "schema_version": "1.0",
  "service": "rds",
  "date": "2025-06-15",
  "environment": "prod",
  "readings": [
    {
      "collection_timestamp": "2025-06-15T10:15:00Z",
      "product": "fgwrds-cluster",
      "metrics": {
        "select_latency": {
          "value": 125.6,
          "unit": "milliseconds",
          "threshold_warning": 50,
          "threshold_critical": 100
        }
      }
    }
  ]
}
```

#### Infrastructure-Specific Keys

| Key | Description | Example |
|-----|-------------|---------|
| `service` | AWS service name | `"rds"`, `"eks"`, `"sqs"` |
| `product` | Specific resource identifier | `"fgwrds-cluster"` |
| `collection_timestamp` | When metrics were collected | `"2025-06-15T10:15:00Z"` |
| `metrics.{name}.value` | Actual metric value | `125.6` |
| `metrics.{name}.unit` | Measurement unit | `"milliseconds"` |
| `metrics.{name}.threshold_warning` | Warning threshold | `50` |
| `metrics.{name}.threshold_critical` | Critical threshold | `100` |

### Business Metrics Schema

#### Markers Format
```json
{
  "schema_version": "1.0",
  "service": "business",
  "date": "2025-06-15",
  "environment": "prod",
  "domain": "markers",
  "readings": [
    {
      "collection_timestamp": "2025-06-15T06:45:00Z",
      "metrics": {
        "GLOBAL-CCCCollateral-EOD_Close": {
          "actual_arrival_time": "2025-06-15T06:45:00Z",
          "expected_arrival_time": "2025-06-15T06:30:00Z",
          "sla_threshold_minutes": 30,
          "arrival_delay_minutes": 15,
          "sla_status": "BREACHED",
          "business_impact": "medium",
          "upstream_source": "CCC_System",
          "data_category": "collateral"
        }
      }
    }
  ]
}
```

#### Milestones Format
```json
{
  "schema_version": "1.0",
  "service": "business",
  "date": "2025-06-15",
  "environment": "prod",
  "domain": "milestones",
  "readings": [
    {
      "collection_timestamp": "2025-06-15T11:20:00Z",
      "process_name": "COLLATERAL",
      "metrics": {
        "start_time": "2025-06-15T09:00:00Z",
        "end_time": "2025-06-15T11:20:00Z",
        "runtime_minutes": 140,
        "sla_completion_time": "2025-06-15T11:00:00Z",
        "completion_delay_minutes": 20,
        "completion_status": "LATE",
        "business_impact": "high",
        "process_category": "risk_management",
        "dependencies": ["GLOBAL-CCCCollateral-EOD_Close"]
      }
    }
  ]
}
```

#### Business-Specific Keys

| Key | Description | Example |
|-----|-------------|---------|
| `domain` | Business metric category | `"markers"`, `"milestones"`, `"sla"` |
| `sla_status` | SLA compliance status | `"MET"`, `"BREACHED"`, `"LATE"` |
| `business_impact` | Impact classification | `"low"`, `"medium"`, `"high"`, `"critical"` |
| `arrival_delay_minutes` | Time past expected arrival | `15` |
| `completion_delay_minutes` | Time past SLA deadline | `20` |
| `upstream_source` | Source system | `"CCC_System"` |
| `dependencies` | Upstream dependencies | `["GLOBAL-CCCCollateral-EOD_Close"]` |

---

## Snowflake Table Creation

### Infrastructure Metrics External Table

```sql
-- Create external table for infrastructure metrics
CREATE OR REPLACE EXTERNAL TABLE infra_metrics_table (
  schema_version STRING,
  service STRING,
  date DATE,
  environment STRING,
  readings VARIANT
)
PARTITION BY (
  DATE_TRUNC('day', date),
  service
)
LOCATION = 's3://metrics-bucket/infra/'
FILE_FORMAT = (TYPE = JSON)
AUTO_REFRESH = TRUE;

-- Add sample partitions (auto-discovered with AUTO_REFRESH)
ALTER TABLE infra_metrics_table 
ADD PARTITION (date='2025-06-15', service='rds') 
LOCATION 's3://metrics-bucket/infra/year=2025/month=06/day=15/service=rds/';

ALTER TABLE infra_metrics_table 
ADD PARTITION (date='2025-06-15', service='sqs') 
LOCATION 's3://metrics-bucket/infra/year=2025/month=06/day=15/service=sqs/';

ALTER TABLE infra_metrics_table 
ADD PARTITION (date='2025-06-15', service='eks') 
LOCATION 's3://metrics-bucket/infra/year=2025/month=06/day=15/service=eks/';
```

### Business Metrics External Table

```sql
-- Create external table for business metrics
CREATE OR REPLACE EXTERNAL TABLE business_metrics_table (
  schema_version STRING,
  service STRING,
  date DATE,
  environment STRING,
  domain STRING,
  readings VARIANT
)
PARTITION BY (
  DATE_TRUNC('day', date),
  domain
)
LOCATION = 's3://metrics-bucket/business/'
FILE_FORMAT = (TYPE = JSON)
AUTO_REFRESH = TRUE;
```

---

## Sample RCA Query

### Cross-Service Root Cause Analysis for Milestone Delays

```sql
-- Comprehensive RCA Query: Milestone delays caused by infrastructure and marker issues
WITH incident_window AS (
  SELECT '2025-06-15 09:00:00'::timestamp as start_time,
         '2025-06-15 12:00:00'::timestamp as end_time
),

-- Extract RDS performance issues
rds_performance AS (
  SELECT 
    r.value:collection_timestamp::timestamp as timestamp,
    r.value:product::string as rds_cluster,
    r.value:metrics:select_latency:value::float as select_latency_ms,
    r.value:metrics:cpu_utilization:value::float as cpu_utilization,
    r.value:metrics:database_connections:value::int as db_connections,
    CASE 
      WHEN r.value:metrics:select_latency:value::float > 100 THEN 'CRITICAL'
      WHEN r.value:metrics:select_latency:value::float > 50 THEN 'WARNING'
      ELSE 'NORMAL'
    END as db_latency_status
  FROM infra_metrics_table imt,
       LATERAL FLATTEN(input => imt.readings) r,
       incident_window iw
  WHERE imt.service = 'rds'
    AND imt.date = '2025-06-15'
    AND r.value:collection_timestamp::timestamp BETWEEN iw.start_time AND iw.end_time
),

-- Extract SQS queue issues
sqs_issues AS (
  SELECT 
    r.value:collection_timestamp::timestamp as timestamp,
    r.value:product::string as sqs_queue,
    r.value:metrics:approximate_age_of_oldest_message:value::int as oldest_message_age_sec,
    r.value:metrics:approximate_number_of_messages_visible:value::int as messages_visible,
    CASE 
      WHEN r.value:metrics:approximate_age_of_oldest_message:value::int > 600 THEN 'CRITICAL'
      WHEN r.value:metrics:approximate_age_of_oldest_message:value::int > 300 THEN 'WARNING'
      ELSE 'NORMAL'
    END as queue_status
  FROM infra_metrics_table imt,
       LATERAL FLATTEN(input => imt.readings) r,
       incident_window iw
  WHERE imt.service = 'sqs'
    AND imt.date = '2025-06-15'
    AND r.value:collection_timestamp::timestamp BETWEEN iw.start_time AND iw.end_time
    AND r.value:product::string LIKE '%lrss-v2-job-notification%'
),

-- Extract marker delays
marker_delays AS (
  SELECT 
    r.value:collection_timestamp::timestamp as marker_timestamp,
    m.key::string as marker_name,
    m.value:actual_arrival_time::timestamp as actual_arrival,
    m.value:expected_arrival_time::timestamp as expected_arrival,
    m.value:arrival_delay_minutes::int as delay_minutes,
    m.value:sla_status::string as marker_sla_status,
    m.value:business_impact::string as marker_impact,
    m.value:data_category::string as data_category
  FROM business_metrics_table bmt,
       LATERAL FLATTEN(input => bmt.readings) r,
       LATERAL FLATTEN(input => r.value:metrics) m,
       incident_window iw
  WHERE bmt.domain = 'markers'
    AND bmt.date = '2025-06-15'
    AND m.value:sla_status::string = 'BREACHED'
    AND r.value:collection_timestamp::timestamp BETWEEN iw.start_time AND iw.end_time
),

-- Extract milestone delays
milestone_delays AS (
  SELECT 
    r.value:collection_timestamp::timestamp as completion_time,
    r.value:process_name::string as process_name,
    r.value:metrics:start_time::timestamp as start_time,
    r.value:metrics:end_time::timestamp as end_time,
    r.value:metrics:completion_delay_minutes::int as completion_delay,
    r.value:metrics:completion_status::string as completion_status,
    r.value:metrics:business_impact::string as business_impact,
    r.value:metrics:dependencies as dependencies
  FROM business_metrics_table bmt,
       LATERAL FLATTEN(input => bmt.readings) r,
       incident_window iw
  WHERE bmt.domain = 'milestones'
    AND bmt.date = '2025-06-15'
    AND r.value:metrics:completion_status::string = 'LATE'
    AND r.value:collection_timestamp::timestamp BETWEEN iw.start_time AND iw.end_time
),

-- Correlate all issues by time proximity
correlated_analysis AS (
  SELECT 
    ms.process_name,
    ms.completion_time,
    ms.completion_delay as milestone_delay_minutes,
    ms.business_impact as milestone_impact,
    ms.dependencies,
    
    -- RDS Issues
    rds.select_latency_ms,
    rds.cpu_utilization,
    rds.db_latency_status,
    
    -- SQS Issues  
    sqs.oldest_message_age_sec,
    sqs.messages_visible,
    sqs.queue_status,
    
    -- Marker Issues
    mk.marker_name,
    mk.delay_minutes as marker_delay_minutes,
    mk.marker_sla_status,
    mk.data_category,
    
    -- Time correlation analysis
    ABS(DATEDIFF('minute', ms.completion_time, rds.timestamp)) as rds_time_diff,
    ABS(DATEDIFF('minute', ms.completion_time, sqs.timestamp)) as sqs_time_diff,
    ABS(DATEDIFF('minute', ms.completion_time, mk.marker_timestamp)) as marker_time_diff
    
  FROM milestone_delays ms
  LEFT JOIN rds_performance rds 
    ON ABS(DATEDIFF('minute', ms.completion_time, rds.timestamp)) <= 60
  LEFT JOIN sqs_issues sqs 
    ON ABS(DATEDIFF('minute', ms.completion_time, sqs.timestamp)) <= 60
  LEFT JOIN marker_delays mk 
    ON ARRAY_CONTAINS(mk.marker_name::VARIANT, ms.dependencies)
)

-- Final RCA Analysis
SELECT 
  process_name,
  completion_time,
  milestone_delay_minutes,
  milestone_impact,
  
  -- Infrastructure Issues
  ROUND(select_latency_ms, 2) as db_latency_ms,
  ROUND(cpu_utilization, 1) as db_cpu_pct,
  db_latency_status,
  
  oldest_message_age_sec as sqs_oldest_msg_sec,
  messages_visible as sqs_queue_size,
  queue_status as sqs_status,
  
  -- Business Dependencies
  marker_name as delayed_marker,
  marker_delay_minutes,
  marker_sla_status,
  data_category,
  
  -- Root Cause Hypothesis
  CASE 
    WHEN db_latency_status = 'CRITICAL' 
         AND queue_status = 'CRITICAL' 
         AND marker_sla_status = 'BREACHED'
    THEN 'CASCADING_FAILURE: DB latency → SQS backup → Marker delay → Milestone delay'
    
    WHEN db_latency_status = 'CRITICAL' 
         AND queue_status = 'CRITICAL'
    THEN 'INFRASTRUCTURE_ISSUE: DB performance causing SQS backup and milestone delay'
    
    WHEN marker_sla_status = 'BREACHED' 
         AND queue_status = 'CRITICAL'
    THEN 'DATA_DEPENDENCY: Late marker arrival causing SQS backup and milestone delay'
    
    WHEN queue_status = 'CRITICAL' 
         AND db_latency_status = 'NORMAL'
    THEN 'SQS_PROCESSING_ISSUE: Queue backup independent of DB performance'
    
    WHEN marker_sla_status = 'BREACHED'
    THEN 'UPSTREAM_DATA_DELAY: Late marker arrival causing milestone delay'
    
    ELSE 'INVESTIGATE_FURTHER: No clear infrastructure or dependency correlation'
  END as root_cause_hypothesis,
  
  -- Correlation Confidence
  CASE 
    WHEN rds_time_diff <= 15 AND sqs_time_diff <= 15 AND marker_time_diff <= 30
    THEN 'HIGH_CONFIDENCE'
    WHEN rds_time_diff <= 30 AND sqs_time_diff <= 30 
    THEN 'MEDIUM_CONFIDENCE'
    ELSE 'LOW_CONFIDENCE'
  END as correlation_confidence

FROM correlated_analysis
ORDER BY completion_time, milestone_delay_minutes DESC;
```

---

## Sample Query Results & Analysis

### Query Results

| process_name | completion_time | milestone_delay_minutes | milestone_impact | db_latency_ms | db_cpu_pct | db_latency_status | sqs_oldest_msg_sec | sqs_queue_size | sqs_status | delayed_marker | marker_delay_minutes | marker_sla_status | data_category | root_cause_hypothesis | correlation_confidence |
|--------------|-----------------|------------------------|------------------|---------------|------------|-------------------|-------------------|----------------|------------|----------------|---------------------|-------------------|---------------|----------------------|----------------------|
| COLLATERAL | 2025-06-15 11:20:00 | 20 | high | 125.6 | 88.2 | CRITICAL | 890 | 1250 | CRITICAL | GLOBAL-CCCCollateral-EOD_Close | 15 | BREACHED | collateral | CASCADING_FAILURE: DB latency → SQS backup → Marker delay → Milestone delay | HIGH_CONFIDENCE |
| DERIVATIVES | 2025-06-15 10:45:00 | 25 | medium | 98.4 | 76.1 | WARNING | 720 | 1100 | CRITICAL | GLOBAL-OTCRegDerivatives-EOD_Close | 30 | BREACHED | derivatives | DATA_DEPENDENCY: Late marker arrival causing SQS backup and milestone delay | MEDIUM_CONFIDENCE |
| SECURITIES | 2025-06-15 12:15:00 | 15 | medium | 45.2 | 62.3 | NORMAL | 450 | 800 | WARNING | GLOBAL-Security-EOD_Close | 5 | BREACHED | securities | UPSTREAM_DATA_DELAY: Late marker arrival causing milestone delay | MEDIUM_CONFIDENCE |

### RCA Analysis Explanation

#### Primary Incident: COLLATERAL Processing Delay

**Timeline & Root Cause Cascade:**

1. **09:00 AM**: COLLATERAL process started
2. **10:15 AM**: Database performance degraded (125.6ms latency, 88.2% CPU)
3. **10:30 AM**: SQS queue backed up (1,250 messages, 890 sec oldest message)
4. **10:45 AM**: GLOBAL-CCCCollateral-EOD_Close marker arrived 15 minutes late
5. **11:20 AM**: COLLATERAL process completed 20 minutes late

**Root Cause Analysis:**

The query results reveal a **cascading failure pattern**:

```
Database Performance Issue (10:15 AM)
        ↓
SQS Queue Backup (10:30 AM)  
        ↓
Delayed Marker Arrival (10:45 AM)
        ↓  
Milestone Processing Delay (11:20 AM)
```

**Detailed Explanation:**

1. **Initial Trigger**: RDS select latency spiked to 125.6ms (25% above critical threshold) with CPU at 88.2%
   - **Impact**: Slower database queries affecting all dependent processes

2. **SQS Backup**: Database slowness caused message processing delays
   - **Evidence**: 1,250 messages queued (25% over capacity), oldest message aging 890 seconds
   - **Impact**: Job notifications and data processing requests backed up

3. **Marker Delay**: Upstream CCC system affected by infrastructure issues
   - **Evidence**: GLOBAL-CCCCollateral-EOD_Close arrived 15 minutes past SLA
   - **Impact**: COLLATERAL process couldn't start full processing without required data

4. **Milestone Delay**: Combined infrastructure and data dependency issues
   - **Result**: COLLATERAL process completed 20 minutes late
   - **Business Impact**: High - affects risk calculations and regulatory reporting

#### Secondary Issues Identified:

- **DERIVATIVES**: Primarily marker-driven delay (30-minute marker delay → 25-minute milestone delay)
- **SECURITIES**: Minor upstream data delay with manageable impact

#### Business Impact Assessment:

| Process | Delay | Business Impact | Root Cause |
|---------|-------|----------------|-------------|
| COLLATERAL | 20 min | **HIGH** - Risk calculations delayed | Cascading infrastructure failure |
| DERIVATIVES | 25 min | **MEDIUM** - Trading desk impact | Upstream data dependency |
| SECURITIES | 15 min | **MEDIUM** - Manageable delay | Minor upstream issue |

#### Preventive Actions:

1. **Database Performance**: Investigate root cause of RDS latency spike
2. **Queue Monitoring**: Implement early warning for SQS backup
3. **Marker SLA**: Engage upstream teams on marker delivery reliability
4. **Correlation Alerts**: Implement cross-service dependency monitoring

This RCA demonstrates how the centralized metric collection platform enables rapid identification of complex, multi-system failure patterns that would be difficult to diagnose through traditional siloed monitoring approaches.