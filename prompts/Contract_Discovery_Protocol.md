# AI-Native Data Product — Contract Discovery Protocol
## Agent Prompt Fragment for Contract-Aware Data Product Access

---

## Purpose

This prompt fragment extends the **AI-Native Data Product Discovery Protocol**
(`Access_Data_Product_Starter.md`) with contract-awareness. Include it alongside
that fragment when an agent needs to understand what an interface promises,
whether it is currently compliant, and what consumers are affected by violations.

This fragment assumes the agent has already completed standard product discovery
(module map, entity metadata, relationships). It adds a contract discovery tier.

---

## Prompt Fragment (Include in Agent System Prompt)

```markdown
## Contract Discovery Protocol

### When to Use

Before consuming any data from a contracted interface, or when asked to:
- Assess whether an interface is safe to use
- Explain what an interface promises
- Identify who is affected by a quality issue
- Check whether data from a specific run was produced from a compliant interface

### Discovery Tier 4: Find Contracted Interfaces

After standard product discovery, query the contract backbone:

```sql
-- What interfaces are contracted, and are they currently compliant?
SELECT
     contract_key
    ,interface_name
    ,interface_description
    ,interface_type
    ,database_name
    ,object_name
    ,validation_status
    ,publication_status
    ,is_compliant
    ,freshness_sla_minutes
    ,last_validated_dts
FROM {Product}_Semantic_V.contract_current_interface
ORDER BY interface_name
;
```

**Interpreting `is_compliant`:**
- `1` = the interface is `PUBLISHED` — data is being served and all CRITICAL rules passed
- `0` = the interface is `NON_COMPLIANT`, `SUSPENDED`, or `UNPUBLISHED` — do not consume

**Interpreting `publication_status`:**
- `PUBLISHED` — safe to consume
- `NON_COMPLIANT` — CRITICAL validation failure; the Access layer view returns no rows until resolved
- `SUSPENDED` — manually suspended; enquire with the contract owner before consuming
- `UNPUBLISHED` — not yet through a validation cycle; treat as unavailable

### Discovery Tier 5: Understand the Contract Promise

To understand what a specific interface promises:

```sql
-- What fields are contracted, their types, and classification?
SELECT
     interface_name
    ,field_name
    ,field_description
    ,data_type
    ,is_nullable
    ,data_classification
    ,is_pii
    ,breaking_change_rule
FROM {Product}_Semantic_V.contract_current_field
WHERE interface_name = '{interface_name}'
ORDER BY field_name
;

-- What quality rules apply to this interface?
SELECT
     interface_name
    ,field_name
    ,rule_name
    ,rule_type
    ,rule_sub_type
    ,rule_description
    ,rule_severity
    ,threshold_value
    ,threshold_operator
    ,threshold_unit
    ,gates_publication
FROM {Product}_Semantic_V.contract_current_rule
WHERE interface_name = '{interface_name}'
ORDER BY rule_severity, rule_type
;
```

### Discovery Tier 6: Check Current Compliance Evidence

To check the latest validation results before consuming:

```sql
-- What was the result of the last validation cycle?
SELECT
     contract_key
    ,interface_name
    ,run_status
    ,rules_total
    ,rules_passed
    ,rules_failed
    ,rules_warned
    ,run_started_dts
    ,run_completed_dts
    ,is_compliant
FROM {Product}_Observability_V.contract_latest_validation
WHERE interface_name = '{interface_name}'
;

-- Are there any open violations I should know about?
SELECT
     interface_name
    ,violation_type
    ,rule_name
    ,rule_severity
    ,violation_message
    ,affected_consumer_count
    ,detected_dts
    ,remediation_status
FROM {Product}_Observability_V.contract_violation_summary
WHERE interface_name = '{interface_name}'
  AND remediation_status = 'OPEN'
ORDER BY
     CASE rule_severity WHEN 'CRITICAL' THEN 1 WHEN 'HIGH' THEN 2
                        WHEN 'MEDIUM' THEN 3 ELSE 4 END
;
```

### Discovery Tier 7: Consumer Impact (If Investigating a Violation)

If investigating whether a violation affects specific consumers:

```sql
-- Who is registered as a consumer of this interface?
SELECT
     interface_name
    ,version_number
    ,consumer_name
    ,consumer_owner
    ,consumer_purpose
    ,data_classification_access
    ,notification_channel
    ,registration_status
FROM {Product}_Observability_V.contract_registered_consumer
WHERE interface_name = '{interface_name}'
  AND registration_status = 'ACTIVE'
;

-- Which consumers are affected by the current open violations?
SELECT
     interface_name
    ,violation_type
    ,rule_severity
    ,consumer_name
    ,consumer_owner
    ,notification_channel
    ,affected_since_dts
    ,remediation_status
FROM {Product}_Observability_V.contract_consumer_impact
WHERE interface_name = '{interface_name}'
;
```

### Key Principles for Contract-Aware Consumption

**1. Always check `is_compliant` before loading data**

If `is_compliant = 0`, the interface may return no rows (BLOCKING mode) or stale/
invalid data (ADVISORY mode). Do not load data from a `NON_COMPLIANT` interface
into a downstream model, report, or production system without explicit approval
from the contract owner.

**2. A view returning no rows may be the contract working correctly**

If a contracted interface returns no rows and `publication_status = 'NON_COMPLIANT'`,
that is the publication gate working as designed — the interface is refusing to serve
data that failed its quality checks. This is not a bug. Notify the contract owner.

**3. Consume the version you registered against**

If you are a registered consumer, consume the contract version recorded in
`contract_consumer.contract_version_id`. If a new version is available, check
`contract_current_interface` for the version number and migrate deliberately.

**4. Use the lineage context view for auditable consumption**

When your consumption of the interface will be recorded in a lineage system,
join your query result to `contract_lineage_context_v` to attach the contract
version and validation status to your lineage event:

```sql
SELECT
     lc.contract_key
    ,lc.version_number         AS contract_version
    ,lc.validation_run_id
    ,lc.validation_run_status
    ,lc.is_compliant
    ,lc.datahub_dataset_urn
FROM {Product}_Semantic_V.contract_lineage_context_v AS lc
WHERE lc.interface_name = '{interface_name}'
;
```

**5. Field classification governs what you may retain**

`contract_field.data_classification` tells you the highest classification level of
each field. Your consuming system's registered `data_classification_access` in
`contract_consumer` must permit that level. Never retain or forward `CONFIDENTIAL`
or `RESTRICTED` fields to a system not authorised for that classification.

### Standard Contract Discovery Sequence

When starting work on a contracted data product:

```sql
-- Step 1: Discover contracted interfaces and their compliance
SELECT interface_name, is_compliant, publication_status
FROM {Product}_Semantic_V.contract_current_interface;

-- Step 2: For each interface you will use — check the promise
SELECT field_name, data_type, is_nullable, data_classification
FROM {Product}_Semantic_V.contract_current_field
WHERE interface_name = '{target_interface}';

-- Step 3: Check latest validation evidence
SELECT run_status, rules_failed, is_compliant
FROM {Product}_Observability_V.contract_latest_validation
WHERE interface_name = '{target_interface}';

-- Step 4: If any violations exist, assess consumer impact
SELECT consumer_name, rule_severity, remediation_status
FROM {Product}_Observability_V.contract_consumer_impact
WHERE interface_name = '{target_interface}';

-- Step 5: Ready to consume (only if is_compliant = 1)
```

### Error Handling

**If `contract_current_interface` does not exist:**
The product may not have been upgraded to the Data Contract Design Standard.
Fall back to standard discovery. The interface has no formal promise — document
this assumption in your run log.

**If `is_compliant = 0` for your target interface:**
Do not consume. Query `contract_violation_summary` to understand why. Notify the
contract owner via the `owner_contact` in `contract_current`. Wait for
`publication_status = 'PUBLISHED'` before consuming.

**If the interface returns no rows despite `is_compliant = 1`:**
This is a data issue, not a contract issue. The contract passed its validation
cycle but the business data itself is empty. Query `contract_latest_validation`
for `rules_passed` and `run_status = 'PASS'` to confirm. Then investigate the
source data pipeline.
```

---

## Notes for Prompt Engineers

**When to include this fragment:**
- Agent will consume data from a product that has the contract backbone deployed
- Agent needs to make trust decisions about data quality before loading
- Agent is performing compliance checks, audit queries, or lineage enrichment
- Agent is investigating a data quality incident

**When NOT to include:**
- Agent is only exploring schema structure (use `Access_Data_Product_Starter.md` alone)
- The product has not yet had the contract backbone added

**Combination with existing protocols:**
Include after the standard discovery tiers in `Access_Data_Product_Starter.md`.
This fragment adds Tiers 4–7 to the existing Tiers 1–3 (module map, entity metadata,
relationships).

**OpenTelemetry span enrichment:**
When the agent is executing inside an instrumented pipeline, the `contract_telemetry_context_v`
view provides the standard OpenTelemetry span attributes. These should be added to any
span that touches a contracted interface:

```sql
SELECT
     contract_key         AS "data_product.contract.id"
    ,contract_version     AS "data_product.contract.version"
    ,is_compliant         AS "data_product.contract.is_compliant"
    ,publication_status   AS "data_product.interface.publication_status"
    ,consumer_count       AS "data_product.contract.consumer_count"
FROM {Product}_Semantic_V.contract_telemetry_context_v
WHERE interface_name = '{interface_name}'
;
```

---

**This prompt fragment is reusable across any AI-Native Data Product deployment
that has the Data Contract Design Standard applied.**
