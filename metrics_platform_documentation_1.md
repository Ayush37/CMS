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

### Cross-Service Root Cause Analysis for Milestone Delays (Using Embedded Thresholds)

```sql
-- Comprehensive RCA Query: Milestone delays caused by infrastructure and marker issues
-- NOTE: This query uses thresholds embedded in JSON data, not hardcoded values
WITH incident_window AS (
  SELECT '2025-06-15 09:00:00'::timestamp as start_time,
         '2025-06-15 12:00:00'::timestamp as end_time
),

-- Extract RDS performance issues using embedded thresholds
rds_performance AS (
  SELECT 
    r.value:collection_timestamp::timestamp as timestamp,
    r.value:product::string as rds_cluster,
    r.value:metrics:select_latency:value::float as select_latency_ms,
    r.value:metrics:select_latency:threshold_warning::float as latency_warning_threshold,
    r.value:metrics:select_latency:threshold_critical::float as latency_critical_threshold,
    r.value:metrics:cpu_utilization:value::float as cpu_utilization,
    r.value:metrics:cpu_utilization:threshold_warning::float as cpu_warning_threshold,
    r.value:metrics:cpu_utilization:threshold_critical::float as cpu_critical_threshold,
    r.value:metrics:database_connections:value::int as db_connections,
    r.value:metrics:database_connections:threshold_critical::int as conn_critical_threshold,
    
    -- Dynamic status calculation using embedded thresholds
    CASE 
      WHEN r.value:metrics:select_latency:value::float > r.value:metrics:select_latency:threshold_critical::float 
      THEN 'CRITICAL'
      WHEN r.value:metrics:select_latency:value::float > r.value:metrics:select_latency:threshold_warning::float 
      THEN 'WARNING'
      ELSE 'NORMAL'
    END as db_latency_status,
    
    CASE 
      WHEN r.value:metrics:cpu_utilization:value::float > r.value:metrics:cpu_utilization:threshold_critical::float 
      THEN 'CRITICAL'
      WHEN r.value:metrics:cpu_utilization:value::float > r.value:metrics:cpu_utilization:threshold_warning::float 
      THEN 'WARNING'
      ELSE 'NORMAL'
    END as cpu_status
  FROM infra_metrics_table imt,
       LATERAL FLATTEN(input => imt.readings) r,
       incident_window iw
  WHERE imt.service = 'rds'
    AND imt.date = '2025-06-15'
    AND r.value:collection_timestamp::timestamp BETWEEN iw.start_time AND iw.end_time
),

-- Extract SQS queue issues using embedded thresholds
sqs_issues AS (
  SELECT 
    r.value:collection_timestamp::timestamp as timestamp,
    r.value:product::string as sqs_queue,
    r.value:metrics:approximate_age_of_oldest_message:value::int as oldest_message_age_sec,
    r.value:metrics:approximate_age_of_oldest_message:threshold_warning::int as msg_age_warning_threshold,
    r.value:metrics:approximate_age_of_oldest_message:threshold_critical::int as msg_age_critical_threshold,
    r.value:metrics:approximate_number_of_messages_visible:value::int as messages_visible,
    r.value:metrics:approximate_number_of_messages_visible:threshold_warning::int as queue_size_warning_threshold,
    r.value:metrics:approximate_number_of_messages_visible:threshold_critical::int as queue_size_critical_threshold,
    
    -- Dynamic status using embedded thresholds
    CASE 
      WHEN r.value:metrics:approximate_age_of_oldest_message:value::int > r.value:metrics:approximate_age_of_oldest_message:threshold_critical::int 
      THEN 'CRITICAL'
      WHEN r.value:metrics:approximate_age_of_oldest_message:value::int > r.value:metrics:approximate_age_of_oldest_message:threshold_warning::int 
      THEN 'WARNING'
      ELSE 'NORMAL'
    END as queue_status,
    
    CASE 
      WHEN r.value:metrics:approximate_number_of_messages_visible:value::int > r.value:metrics:approximate_number_of_messages_visible:threshold_critical::int 
      THEN 'CRITICAL'
      WHEN r.value:metrics:approximate_number_of_messages_visible:value::int > r.value:metrics:approximate_number_of_messages_visible:threshold_warning::int 
      THEN 'WARNING'
      ELSE 'NORMAL'
    END as queue_backlog_status
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
    
    -- RDS Issues with thresholds
    rds.select_latency_ms,
    rds.latency_critical_threshold,
    rds.cpu_utilization,
    rds.cpu_critical_threshold,
    rds.db_connections,
    rds.conn_critical_threshold,
    rds.db_latency_status,
    rds.cpu_status,
    
    -- SQS Issues with thresholds
    sqs.oldest_message_age_sec,
    sqs.msg_age_critical_threshold,
    sqs.messages_visible,
    sqs.queue_size_critical_threshold,
    sqs.queue_status,
    sqs.queue_backlog_status,
    
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
    AND rds.db_latency_status != 'NORMAL'
  LEFT JOIN sqs_issues sqs 
    ON ABS(DATEDIFF('minute', ms.completion_time, sqs.timestamp)) <= 60
    AND (sqs.queue_status != 'NORMAL' OR sqs.queue_backlog_status != 'NORMAL')
  LEFT JOIN marker_delays mk 
    ON ARRAY_CONTAINS(mk.marker_name::VARIANT, ms.dependencies)
)

-- Final RCA Analysis
SELECT 
  process_name,
  completion_time,
  milestone_delay_minutes,
  milestone_impact,
  
  -- Infrastructure Issues with Dynamic Thresholds
  ROUND(select_latency_ms, 2) as db_latency_ms,
  latency_critical_threshold as db_latency_sla,
  ROUND(((select_latency_ms - latency_critical_threshold) / latency_critical_threshold) * 100, 1) as db_latency_breach_pct,
  
  ROUND(cpu_utilization, 1) as db_cpu_pct,
  cpu_critical_threshold as db_cpu_sla,
  db_latency_status,
  cpu_status,
  
  oldest_message_age_sec as sqs_oldest_msg_sec,
  msg_age_critical_threshold as sqs_msg_age_sla,
  messages_visible as sqs_queue_size,
  queue_size_critical_threshold as sqs_queue_size_sla,
  ROUND(((messages_visible - queue_size_critical_threshold) / queue_size_critical_threshold) * 100, 1) as sqs_queue_breach_pct,
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

### Query Results Using Embedded Thresholds

| process_name | completion_time | milestone_delay_minutes | db_latency_ms | db_latency_sla | db_latency_breach_pct | db_cpu_pct | db_cpu_sla | sqs_oldest_msg_sec | sqs_msg_age_sla | sqs_queue_size | sqs_queue_size_sla | sqs_queue_breach_pct | delayed_marker | marker_delay_minutes | root_cause_hypothesis | correlation_confidence |
|--------------|-----------------|------------------------|---------------|----------------|----------------------|------------|------------|-------------------|----------------|----------------|-------------------|--------------------|--------------|--------------------|----------------------|----------------------|
| COLLATERAL | 2025-06-15 11:20:00 | 20 | 125.6 | 100 | 25.6 | 88.2 | 85 | 890 | 600 | 1250 | 1000 | 25.0 | GLOBAL-CCCCollateral-EOD_Close | 15 | CASCADING_FAILURE: DB latency → SQS backup → Marker delay → Milestone delay | HIGH_CONFIDENCE |
| DERIVATIVES | 2025-06-15 10:45:00 | 25 | 98.4 | 100 | -1.6 | 76.1 | 85 | 720 | 600 | 1100 | 1000 | 10.0 | GLOBAL-OTCRegDerivatives-EOD_Close | 30 | DATA_DEPENDENCY: Late marker arrival causing SQS backup and milestone delay | MEDIUM_CONFIDENCE |
| SECURITIES | 2025-06-15 12:15:00 | 15 | 45.2 | 100 | -54.8 | 62.3 | 85 | 450 | 600 | 800 | 1000 | -20.0 | GLOBAL-Security-EOD_Close | 5 | UPSTREAM_DATA_DELAY: Late marker arrival causing milestone delay | MEDIUM_CONFIDENCE |

### RCA Analysis Explanation

#### Primary Incident: COLLATERAL Processing Delay

**Timeline & Root Cause Cascade:**

1. **09:00 AM**: COLLATERAL process started
2. **10:15 AM**: Database performance degraded (125.6ms latency vs 100ms SLA = 25.6% breach, 88.2% CPU vs 85% SLA)
3. **10:30 AM**: SQS queue backed up (1,250 messages vs 1,000 SLA = 25% breach, 890 sec oldest message vs 600 sec SLA)
4. **10:45 AM**: GLOBAL-CCCCollateral-EOD_Close marker arrived 15 minutes late
5. **11:20 AM**: COLLATERAL process completed 20 minutes late

**Root Cause Analysis:**

The query results using **embedded thresholds** reveal a more precise cascading failure pattern:

```
Database Performance Issue (10:15 AM)
   • 125.6ms latency (25.6% above 100ms SLA)
   • 88.2% CPU (3.2% above 85% SLA)
        ↓
SQS Queue Backup (10:30 AM)  
   • 1,250 messages (25% above 1,000 message SLA)
   • 890 second oldest message (48% above 600 sec SLA)
        ↓
Delayed Marker Arrival (10:45 AM)
   • GLOBAL-CCCCollateral-EOD_Close: 15 minutes late
        ↓  
Milestone Processing Delay (11:20 AM)
   • COLLATERAL process: 20 minutes late
```

**Key Advantages of Using Embedded Thresholds:**

1. **Precise Breach Measurement**: Instead of generic "CRITICAL/WARNING", we see exact percentages (25.6% latency breach, 25% queue breach)

2. **Dynamic SLA Management**: Each service can have different thresholds:
   - RDS cluster threshold: 100ms (production standard)
   - SQS queue threshold: 1,000 messages (business capacity)
   - Message age threshold: 600 seconds (processing SLA)

3. **Historical Threshold Tracking**: Query shows what SLAs were in effect during the incident

4. **Threshold Evolution**: When thresholds change, historical analysis remains accurate

**Enhanced Business Impact Assessment:**

| Process | Delay | SLA Breach Analysis | Root Cause |
|---------|-------|-------------------|-------------|
| **COLLATERAL** | 20 min | DB: 25.6% over SLA, Queue: 25% over capacity | **PRIMARY**: Cascading infrastructure failure |
| **DERIVATIVES** | 25 min | DB: Within SLA (-1.6%), Queue: 10% over capacity | **SECONDARY**: Marker dependency with minor infrastructure impact |
| **SECURITIES** | 15 min | DB: 54.8% under SLA, Queue: 20% under capacity | **ISOLATED**: Pure upstream data delay |

#### Preventive Actions:

1. **Database Performance**: Investigate root cause of RDS latency spike
2. **Queue Monitoring**: Implement early warning for SQS backup
3. **Marker SLA**: Engage upstream teams on marker delivery reliability
4. **Correlation Alerts**: Implement cross-service dependency monitoring

This RCA demonstrates how the centralized metric collection platform enables rapid identification of complex, multi-system failure patterns that would be difficult to diagnose through traditional siloed monitoring approaches.