# AI-Native Data Product — Contract Compliance Validation Prompt
## For Investigating, Running, and Triaging Contract Validation

---

## How to Use This Prompt

1. Copy this entire prompt
2. Replace all `[PLACEHOLDER]` values with your specifics
3. Paste into a new Claude conversation with Teradata MCP access
4. Choose the operation that matches your current situation

---

## Purpose

This prompt covers three operational scenarios:

| Scenario | When to use |
|----------|------------|
| **A. Run a validation cycle** | You want to execute the contract rules for one or more interfaces and update publication status |
| **B. Triage an open violation** | An alert fired or a consumer reported an issue; you need to understand impact and guide remediation |
| **C. Audit compliance history** | You need evidence for a regulatory submission, audit, or compliance review |

---

## Starter Prompt

You are a Data Operations specialist working with the **[DATA_PRODUCT_NAME]** data product
on the **[ENVIRONMENT]** Teradata system (host: `[HOST]`).

The product follows the AI-Native Data Product architecture with the Data Contract
Design Standard applied. Contract backbone tables are in:
- `[DATA_PRODUCT_NAME]_Semantic` — contract definitions
- `[DATA_PRODUCT_NAME]_Observability` — validation evidence and violations
- `[DATA_PRODUCT_NAME]_Memory` — governance history

Choose the operation below that matches your task.

---

## Operation A — Run a Validation Cycle

Use this when you want to check contract compliance for one or more interfaces.

### Step A1: Identify what to validate

```sql
-- Which interfaces are due for validation or have not been validated recently?
SELECT
     ci.interface_name
    ,ci.publication_status
    ,ci.validation_status
    ,ci.freshness_sla_minutes
    ,ci.last_validated_dts
    ,CAST(
         (CURRENT_TIMESTAMP(0) - ci.last_validated_dts HOUR(4))
         AS INTEGER
     ) * 60 AS minutes_since_last_validation
FROM [DATA_PRODUCT_NAME]_Semantic.contract_interface AS ci
WHERE ci.is_active = 1
ORDER BY minutes_since_last_validation DESC
;
```

### Step A2: For each interface, execute each contract rule

For each row in `contract_rule` where `is_enforced = 1` for the target interface,
run the validation check defined in `rule_logic` and record results:

```sql
-- See all rules to execute for a specific interface
SELECT
     cr.rule_name
    ,cr.rule_type
    ,cr.rule_sub_type
    ,cr.rule_severity
    ,cr.rule_logic
    ,cr.threshold_value
    ,cr.threshold_operator
    ,cr.threshold_unit
    ,cr.is_enforced
    ,cr.gates_publication
FROM [DATA_PRODUCT_NAME]_Semantic.contract_rule AS cr
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_interface AS ci
    ON ci.contract_interface_id = cr.contract_interface_id
   AND ci.is_active = 1
WHERE ci.interface_name = '[INTERFACE_NAME]'
  AND cr.is_active = 1
ORDER BY cr.rule_severity, cr.rule_type
;
```

### Step A3: Record the validation run

```sql
-- Open a validation run record (insert at start of cycle)
INSERT INTO [DATA_PRODUCT_NAME]_Observability.contract_validation_run
(
     contract_interface_id
    ,run_status
    ,rules_total
    ,rules_passed
    ,rules_failed
    ,rules_warned
    ,run_started_dts
    ,triggered_by
)
SELECT
     ci.contract_interface_id
    ,'RUNNING'
    ,(SELECT COUNT(*) FROM [DATA_PRODUCT_NAME]_Semantic.contract_rule AS cr2
      WHERE cr2.contract_interface_id = ci.contract_interface_id AND cr2.is_active = 1)
    ,0, 0, 0
    ,CURRENT_TIMESTAMP(6)
    ,'[TRIGGERED_BY]'   -- pipeline name, agent ID, or manual
FROM [DATA_PRODUCT_NAME]_Semantic.contract_interface AS ci
WHERE ci.interface_name = '[INTERFACE_NAME]'
;
```

### Step A4: After executing each rule, record results

```sql
-- Record one result per rule (repeat for each rule executed)
INSERT INTO [DATA_PRODUCT_NAME]_Observability.contract_validation_result
(
     validation_run_id
    ,contract_rule_id
    ,result_status       -- 'PASS', 'FAIL', 'WARN', 'SKIP'
    ,measured_value
    ,expected_value
    ,result_message
)
VALUES
(
     [VALIDATION_RUN_ID]
    ,[CONTRACT_RULE_ID]
    ,'[RESULT_STATUS]'
    ,[MEASURED_VALUE]
    ,[EXPECTED_VALUE]
    ,'[RESULT_MESSAGE]'
)
;
```

### Step A5: Update publication status and close the run

```sql
-- Update publication_status based on CRITICAL rule results
UPDATE [DATA_PRODUCT_NAME]_Semantic.contract_interface
SET
     publication_status      =
         CASE
             WHEN publication_gate_mode = 'BLOCKING'
              AND EXISTS (
                  SELECT 1
                  FROM [DATA_PRODUCT_NAME]_Observability.contract_validation_result AS vr
                  INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_rule AS cr
                      ON cr.contract_rule_id = vr.contract_rule_id
                  WHERE vr.validation_run_id = [VALIDATION_RUN_ID]
                    AND cr.rule_severity = 'CRITICAL'
                    AND cr.gates_publication = 1
                    AND vr.result_status = 'FAIL'
              )
             THEN 'NON_COMPLIANT'
             ELSE 'PUBLISHED'
         END
    ,non_compliant_since_dts =
         CASE
             WHEN publication_gate_mode = 'BLOCKING'
              AND EXISTS (
                  SELECT 1
                  FROM [DATA_PRODUCT_NAME]_Observability.contract_validation_result AS vr
                  INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_rule AS cr
                      ON cr.contract_rule_id = vr.contract_rule_id
                  WHERE vr.validation_run_id = [VALIDATION_RUN_ID]
                    AND cr.rule_severity = 'CRITICAL'
                    AND cr.gates_publication = 1
                    AND vr.result_status = 'FAIL'
              )
             THEN CURRENT_TIMESTAMP(6)
             ELSE NULL
         END
    ,validation_status       =
         CASE
             WHEN EXISTS (
                  SELECT 1
                  FROM [DATA_PRODUCT_NAME]_Observability.contract_validation_result AS vr
                  WHERE vr.validation_run_id = [VALIDATION_RUN_ID]
                    AND vr.result_status = 'FAIL'
              )
             THEN 'FAILING'
             WHEN EXISTS (
                  SELECT 1
                  FROM [DATA_PRODUCT_NAME]_Observability.contract_validation_result AS vr
                  WHERE vr.validation_run_id = [VALIDATION_RUN_ID]
                    AND vr.result_status = 'WARN'
              )
             THEN 'WARNING'
             ELSE 'PASSING'
         END
    ,last_validated_dts = CURRENT_TIMESTAMP(6)
WHERE interface_name = '[INTERFACE_NAME]'
;
```

### Step A6: Check the outcome

```sql
-- Confirm final status
SELECT
     interface_name
    ,publication_status
    ,validation_status
    ,is_compliant  -- from the view (derived)
    ,last_validated_dts
FROM [DATA_PRODUCT_NAME]_Semantic_V.contract_current_interface
WHERE interface_name = '[INTERFACE_NAME]'
;
```

---

## Operation B — Triage an Open Violation

Use this when an alert has fired or a consumer has reported a data quality issue.

### Step B1: Assess the violation

```sql
-- What violations are currently open?
SELECT
     v.contract_violation_id
    ,ci.interface_name
    ,v.violation_type
    ,v.rule_name
    ,v.violation_severity
    ,v.violation_message
    ,v.affected_consumer_count
    ,v.detected_dts
    ,v.remediation_status
    ,v.remediation_owner
FROM [DATA_PRODUCT_NAME]_Observability.contract_violation AS v
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_interface AS ci
    ON ci.contract_interface_id = v.contract_interface_id
WHERE v.remediation_status IN ('OPEN', 'IN_PROGRESS')
ORDER BY
     CASE v.violation_severity WHEN 'CRITICAL' THEN 1 WHEN 'HIGH' THEN 2
                               WHEN 'MEDIUM' THEN 3 ELSE 4 END
    ,v.detected_dts
;
```

### Step B2: Identify affected consumers

```sql
-- Which consumers are affected by the open violations?
SELECT
     ci.interface_name
    ,v.violation_severity
    ,v.violation_message
    ,cc.consumer_name
    ,cc.consumer_owner
    ,cc.consumer_contact
    ,cc.notification_channel
    ,cc.consumer_purpose
FROM [DATA_PRODUCT_NAME]_Observability.contract_violation AS v
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_interface AS ci
    ON ci.contract_interface_id = v.contract_interface_id
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_consumer AS cc
    ON cc.contract_interface_id = ci.contract_interface_id
   AND cc.registration_status = 'ACTIVE'
WHERE v.remediation_status = 'OPEN'
ORDER BY v.violation_severity, cc.consumer_name
;
```

### Step B3: Assess pipeline impact

```sql
-- What is the current publication status (is the gate active)?
SELECT
     interface_name
    ,publication_status
    ,publication_gate_mode
    ,non_compliant_since_dts
    ,validation_status
FROM [DATA_PRODUCT_NAME]_Semantic.contract_interface
WHERE interface_name = '[INTERFACE_NAME]'
;
```

If `publication_status = 'NON_COMPLIANT'`:
- The Access layer view is returning no rows to consumers
- Consumers should be notified immediately via their `notification_channel`
- The remediation path is to fix the root cause and re-run the validation cycle (Operation A)

### Step B4: Record investigation progress

```sql
-- Update violation to IN_PROGRESS with owner assigned
UPDATE [DATA_PRODUCT_NAME]_Observability.contract_violation
SET
     remediation_status = 'IN_PROGRESS'
    ,remediation_owner  = '[YOUR_NAME_OR_TEAM]'
    ,remediation_notes  = '[BRIEF_DESCRIPTION_OF_INVESTIGATION]'
WHERE contract_violation_id = [VIOLATION_ID]
;
```

### Step B5: After fixing the root cause, close the violation

```sql
-- Close the violation after root cause is resolved
UPDATE [DATA_PRODUCT_NAME]_Observability.contract_violation
SET
     remediation_status    = 'RESOLVED'
    ,remediation_notes     = '[FULL_DESCRIPTION_OF_FIX]'
    ,resolution_dts        = CURRENT_TIMESTAMP(6)
WHERE contract_violation_id = [VIOLATION_ID]
;
```

Then re-run Operation A to execute the validation cycle and confirm the interface
returns to `PUBLISHED` status.

---

## Operation C — Audit Compliance History

Use this when you need evidence for a regulatory review, audit, or formal submission.

### Step C1: Pull the compliance history for a date range

```sql
-- Validation history for an interface over a period
SELECT
     vr.validation_run_id
    ,ci.interface_name
    ,cv.version_number
    ,vr.run_status
    ,vr.rules_total
    ,vr.rules_passed
    ,vr.rules_failed
    ,vr.run_started_dts
    ,vr.run_completed_dts
    ,vr.triggered_by
FROM [DATA_PRODUCT_NAME]_Observability.contract_validation_run AS vr
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_interface AS ci
    ON ci.contract_interface_id = vr.contract_interface_id
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_version AS cv
    ON cv.contract_version_id = ci.contract_version_id
WHERE ci.interface_name = '[INTERFACE_NAME]'
  AND vr.run_started_dts BETWEEN '[FROM_DATE]' AND '[TO_DATE]'
ORDER BY vr.run_started_dts DESC
;
```

### Step C2: Pull violation history

```sql
-- All violations (open and resolved) for an interface over a period
SELECT
     v.violation_type
    ,v.rule_name
    ,v.violation_severity
    ,v.violation_message
    ,v.affected_consumer_count
    ,v.detected_dts
    ,v.remediation_status
    ,v.resolution_dts
    ,v.remediation_notes
FROM [DATA_PRODUCT_NAME]_Observability.contract_violation AS v
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_interface AS ci
    ON ci.contract_interface_id = v.contract_interface_id
WHERE ci.interface_name = '[INTERFACE_NAME]'
  AND v.detected_dts BETWEEN '[FROM_DATE]' AND '[TO_DATE]'
ORDER BY v.detected_dts DESC
;
```

### Step C3: Pull the contract change log

```sql
-- What changed in this contract over the audit period?
SELECT
     c.contract_key
    ,cv.version_number
    ,cl.change_type
    ,cl.change_description
    ,cl.changed_by
    ,cl.change_dts
    ,cl.breaking_change_ind
    ,cl.migration_notes
FROM [DATA_PRODUCT_NAME]_Memory.contract_change_log AS cl
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_version AS cv
    ON cv.contract_version_id = cl.contract_version_id
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract AS c
    ON c.contract_id = cv.contract_id
WHERE c.contract_key LIKE '%[INTERFACE_NAME]%'
  AND cl.change_dts BETWEEN '[FROM_DATE]' AND '[TO_DATE]'
ORDER BY cl.change_dts
;
```

### Step C4: Pull consumer sign-off records

```sql
-- What approvals were obtained for MAJOR version changes?
SELECT
     c.contract_key
    ,cv.version_number
    ,cc.consumer_name
    ,cs.signoff_status
    ,cs.requested_dts
    ,cs.signoff_dts
    ,cs.signed_off_by
    ,cs.signoff_notes
    ,cs.deadline_dt
FROM [DATA_PRODUCT_NAME]_Memory.contract_consumer_signoff AS cs
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_version AS cv
    ON cv.contract_version_id = cs.contract_version_id
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract AS c
    ON c.contract_id = cv.contract_id
INNER JOIN [DATA_PRODUCT_NAME]_Semantic.contract_consumer AS cc
    ON cc.contract_consumer_id = cs.contract_consumer_id
WHERE c.contract_key LIKE '%[INTERFACE_NAME]%'
ORDER BY cs.requested_dts
;
```

### Step C5: Produce an evidence summary

Summarise the audit findings using the above query results:

1. **Validation pass rate** — `rules_passed / rules_total` over the audit period
2. **SLA compliance rate** — proportion of validation cycles where no `FRESHNESS_MAX_AGE`
   rule failed
3. **Violation count by severity** — breakdown of open vs resolved violations
4. **Average time to resolution** — `AVG(resolution_dts - detected_dts)` for resolved violations
5. **Contract version in force** — confirm which version governed data during the period
6. **Consumer sign-off completeness** — confirm all MAJOR changes had sign-off before taking effect

---

## Escalation Paths

| Situation | Action |
|-----------|--------|
| CRITICAL violation, `NON_COMPLIANT`, consumers affected | Notify contract owner immediately via `owner_contact`; notify affected consumers via `notification_channel`; open a ServiceNow incident |
| CRITICAL violation, `ADVISORY` mode, consumers still receiving data | Notify contract owner; consider temporarily switching to `BLOCKING` mode if data is materially incorrect |
| Violation unresolved beyond SLA (see Section 8.1 of the standard) | Escalate to the contract owner's manager; add `remediation_status = 'ESCALATED'` note |
| Consumer still on `DEPRECATED` version | Contact the consumer via `consumer_contact`; create a `contract_design_decision` record documenting the migration timeline |
