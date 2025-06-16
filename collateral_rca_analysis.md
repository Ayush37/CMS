# COLLATERAL Milestone Delay - RCA Analysis

## Scenario Setup

From our sample data:
- **COLLATERAL process**: Started 09:00, completed 11:20 (20 minutes late)
- **Expected completion**: 11:00 
- **Dependencies**: `["GLOBAL-CCCCollateral-EOD_Close"]`
- **Incident window**: 09:00 - 12:00

---

## Step-by-Step Query Execution for COLLATERAL

### 1. Incident Window Definition
```sql
incident_window: start_time='2025-06-15 09:00:00', end_time='2025-06-15 12:00:00'
```
**Purpose**: 3-hour investigation window covering COLLATERAL processing period

---

### 2. RDS Performance Analysis CTE - COLLATERAL Timeline

**What the query scans**:
```
RDS JSON file readings array (15-minute intervals):
09:00 → select_latency: 45ms, cpu: 65% [NORMAL]
09:15 → select_latency: 52ms, cpu: 68% [WARNING] 
09:30 → select_latency: 48ms, cpu: 72% [NORMAL]
09:45 → select_latency: 78ms, cpu: 75% [WARNING]
10:00 → select_latency: 89ms, cpu: 82% [WARNING]
10:15 → select_latency: 125.6ms, cpu: 88.2% [CRITICAL] ← BREACH!
10:30 → select_latency: 132ms, cpu: 90% [CRITICAL] ← BREACH!
10:45 → select_latency: 118ms, cpu: 85.1% [CRITICAL] ← BREACH!
11:00 → select_latency: 95ms, cpu: 78% [WARNING]
11:15 → select_latency: 72ms, cpu: 71% [WARNING]
11:30 → select_latency: 58ms, cpu: 68% [NORMAL]
11:45 → select_latency: 45ms, cpu: 64% [NORMAL]
```

**RDS Performance CTE Returns**:
```sql
-- Three rows showing the breach period
Row 1: timestamp='10:15', select_latency_ms=125.6, db_latency_status='CRITICAL'
Row 2: timestamp='10:30', select_latency_ms=132.0, db_latency_status='CRITICAL' 
Row 3: timestamp='10:45', select_latency_ms=118.0, db_latency_status='CRITICAL'
```

---

### 3. SQS Issues Analysis CTE - COLLATERAL Timeline

**What the query scans**:
```
SQS JSON file readings (job notification queue):
09:00 → messages: 145, oldest_age: 120sec [NORMAL]
09:15 → messages: 189, oldest_age: 180sec [NORMAL]
09:30 → messages: 298, oldest_age: 250sec [NORMAL]
09:45 → messages: 456, oldest_age: 290sec [NORMAL]
10:00 → messages: 678, oldest_age: 380sec [WARNING]
10:15 → messages: 892, oldest_age: 480sec [WARNING]
10:30 → messages: 1250, oldest_age: 890sec [CRITICAL] ← BREACH!
10:45 → messages: 1456, oldest_age: 1080sec [CRITICAL] ← BREACH!
11:00 → messages: 1123, oldest_age: 890sec [CRITICAL] ← BREACH!
11:15 → messages: 865, oldest_age: 720sec [WARNING]
11:30 → messages: 567, oldest_age: 450sec [NORMAL]
```

**SQS Issues CTE Returns**:
```sql
-- Three rows showing queue backup period
Row 1: timestamp='10:30', messages_visible=1250, queue_status='CRITICAL'
Row 2: timestamp='10:45', messages_visible=1456, queue_status='CRITICAL'
Row 3: timestamp='11:00', messages_visible=1123, queue_status='CRITICAL'
```

---

### 4. Marker Delays Analysis CTE - COLLATERAL Dependencies

**What the query scans**:
```
Business markers JSON for breached markers:
GLOBAL-CCCCollateral-EOD_Close:
  - expected_arrival: '06:30'
  - actual_arrival: '06:45' 
  - delay: 15 minutes
  - sla_status: 'BREACHED'
  - marker_timestamp: '06:45' (when breach was recorded)
```

**Marker Delays CTE Returns**:
```sql
Row 1: marker_name='GLOBAL-CCCCollateral-EOD_Close', 
       delay_minutes=15, 
       marker_sla_status='BREACHED',
       marker_timestamp='06:45'
```

---

### 5. Milestone Delays Analysis CTE

**What the query scans**:
```
Business milestones JSON for late processes:
COLLATERAL:
  - start_time: '09:00'
  - end_time: '11:20'
  - sla_completion_time: '11:00'
  - completion_delay_minutes: 20
  - completion_status: 'LATE'
  - dependencies: ['GLOBAL-CCCCollateral-EOD_Close']
```

**Milestone Delays CTE Returns**:
```sql
Row 1: process_name='COLLATERAL',
       completion_time='11:20',
       completion_delay=20,
       dependencies=['GLOBAL-CCCCollateral-EOD_Close']
```

---

### 6. The Correlation Analysis - COLLATERAL Timeline

**Correlation Logic**:
```sql
-- COLLATERAL completed at 11:20
-- Find infrastructure issues within 60 minutes of 11:20

-- RDS breaches: 10:15, 10:30, 10:45
-- Time differences from 11:20:
--   10:15 → 65 minutes (outside 60-min window)
--   10:30 → 50 minutes (within window) ✓
--   10:45 → 35 minutes (within window) ✓

-- SQS breaches: 10:30, 10:45, 11:00  
-- Time differences from 11:20:
--   10:30 → 50 minutes (within window) ✓
--   10:45 → 35 minutes (within window) ✓
--   11:00 → 20 minutes (within window) ✓

-- Marker breach: 06:45
-- Time difference from 11:20:
--   06:45 → 275 minutes (way outside window, but...)
--   Special logic: ARRAY_CONTAINS check for dependencies
--   COLLATERAL depends on 'GLOBAL-CCCCollateral-EOD_Close' ✓
```

**Correlated Analysis Returns**:
```sql
-- Multiple correlation rows (one per infrastructure issue combination)
Row 1: COLLATERAL + RDS(10:30) + SQS(10:30) + Marker(06:45)
       rds_time_diff=50, sqs_time_diff=50, marker_time_diff=275
       
Row 2: COLLATERAL + RDS(10:45) + SQS(10:45) + Marker(06:45)  
       rds_time_diff=35, sqs_time_diff=35, marker_time_diff=275
```

---

## Final RCA Analysis Output for COLLATERAL

### Primary Correlation Result:
```sql
process_name: COLLATERAL
completion_time: 2025-06-15 11:20:00
milestone_delay_minutes: 20
milestone_impact: high

-- Infrastructure Issues (Best Correlation - 10:30 timeframe)
db_latency_ms: 132.0
db_latency_sla: 100  
db_latency_breach_pct: 32.0%    -- 32% over SLA
db_cpu_pct: 90.0
db_cpu_sla: 85
cpu_status: CRITICAL

sqs_oldest_msg_sec: 890
sqs_msg_age_sla: 600
sqs_queue_size: 1250  
sqs_queue_size_sla: 1000
sqs_queue_breach_pct: 25.0%     -- 25% over capacity
sqs_status: CRITICAL

-- Business Dependencies  
delayed_marker: GLOBAL-CCCCollateral-EOD_Close
marker_delay_minutes: 15
marker_sla_status: BREACHED
data_category: collateral

-- Time Correlations
rds_time_diff: 50               -- 50 minutes before completion
sqs_time_diff: 50               -- 50 minutes before completion  
marker_time_diff: 275           -- 275 minutes before completion

-- RCA Conclusion
root_cause_hypothesis: CASCADING_FAILURE: DB latency → SQS backup → Marker delay → Milestone delay
correlation_confidence: HIGH_CONFIDENCE  -- Both RDS and SQS within 60min window
```

---

## Detailed Timeline Analysis for COLLATERAL

### The Cascading Failure Pattern:

```
06:45 AM │ Marker Delay (Root Cause)
         │ GLOBAL-CCCCollateral-EOD_Close arrives 15 minutes late
         │ └─ Upstream CCC system issue
         │
09:00 AM │ COLLATERAL Process Starts
         │ └─ Starts with incomplete/delayed data dependency
         │
10:15 AM │ Database Performance Degradation Begins  
         │ └─ 125.6ms latency (25.6% over SLA)
         │ └─ Caused by: Processing delayed marker data + normal workload
         │
10:30 AM │ Infrastructure Crisis Peak
         │ ├─ DB: 132ms latency (32% over SLA), 90% CPU
         │ └─ SQS: 1,250 messages (25% over capacity), 890sec aging
         │     └─ Queue backup caused by slow DB queries
         │
10:45 AM │ Continued Degradation
         │ ├─ DB: Still critical (118ms latency)  
         │ └─ SQS: Peak queue size (1,456 messages)
         │
11:00 AM │ Infrastructure Recovery Begins
         │ ├─ DB: Latency improves to 95ms (warning level)
         │ └─ SQS: Queue starts draining (1,123 messages)
         │
11:20 AM │ COLLATERAL Completes (20 minutes late)
         │ └─ Late due to: Initial marker delay + infrastructure slowdown
```

---

## RCA Intelligence Extracted

### 1. Multi-Factor Root Cause:
- **Primary Trigger**: Late marker arrival (06:45) → incomplete data dependency
- **Amplification**: Database performance degradation (10:15-11:00) → slow processing  
- **Cascade Effect**: SQS queue backup (10:30-11:00) → job processing delays

### 2. Time-Based Correlation Confidence:
```sql
-- HIGH_CONFIDENCE because:
-- rds_time_diff=50 ≤ 60 ✓
-- sqs_time_diff=50 ≤ 60 ✓  
-- Both infrastructure issues within correlation window
```

### 3. Business Impact Calculation:
```sql
-- 20-minute delay on HIGH impact process
-- 32% database SLA breach during critical processing window  
-- 25% queue capacity breach causing processing bottleneck
-- Dependencies on breached upstream marker
```

### 4. Precise Breach Quantification:
- **Database**: 32% over performance SLA during peak processing
- **Queue**: 25% over capacity SLA causing bottleneck  
- **Marker**: 15-minute dependency delay at process start
- **Overall**: 20-minute milestone SLA breach

## Key Insights

**The query successfully identifies**:
1. **Temporal clustering** of infrastructure issues during COLLATERAL processing
2. **Quantified impact** of each infrastructure breach on business operations
3. **Dependency chain** from marker delay → processing delay → infrastructure stress
4. **Recovery timeline** showing when systems returned to normal

**This demonstrates how a brief infrastructure degradation (45 minutes) combined with upstream dependency issues can cause significant business process delays, providing clear, data-driven RCA for operational improvement.**