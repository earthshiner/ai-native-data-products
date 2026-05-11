# Data Contract Design Standard
## AI-Native Data Product Architecture - Version 1.0

---

## Document Control

| Attribute | Value |
|-----------|-------|
| **Version** | 1.0 |
| **Status** | DRAFT |
| **Last Updated** | 2026-05-11 |
| **Owner** | Worldwide Data Architecture Team, Teradata |
| **Scope** | Data Contract Design Standard (Cross-Cutting) |
| **Type** | Design Standard (Cross-Cutting — not a module) |

---

## Table of Contents

1. [Overview and Purpose](#1-overview-and-purpose)
2. [Fundamental Principle: Cross-Cutting, Not a New Module](#2-fundamental-principle-cross-cutting-not-a-new-module)
3. [Mandatory Contract Rule](#3-mandatory-contract-rule)
4. [Contract Table Design](#4-contract-table-design)
5. [Contract Lifecycle](#5-contract-lifecycle)
6. [Semantic Versioning and Breaking-Change Rules](#6-semantic-versioning-and-breaking-change-rules)
7. [Consumer Registration and Sign-Off](#7-consumer-registration-and-sign-off)
8. [Violation Events and Alerting](#8-violation-events-and-alerting)
9. [Access, Roles, and Grants Tied to Contracts](#9-access-roles-and-grants-tied-to-contracts)
10. [External Standards Integration](#10-external-standards-integration)
11. [Repository and File Structure](#11-repository-and-file-structure)
12. [Contract View Groups](#12-contract-view-groups)
13. [Interactive Contract Dashboards](#13-interactive-contract-dashboards)
14. [Multi-Consumer View Pattern](#14-multi-consumer-view-pattern)
15. [Designer Responsibilities](#15-designer-responsibilities)

---

> **Platform Note:** All DDL and SQL examples in this standard use **Teradata** syntax (e.g. `REPLACE VIEW`, `LOCKING ROW FOR ACCESS`, uppercase keywords). When implementing on a different database platform, these examples must be converted to the target vendor's equivalent DDL. The structural design — table schemas, column contracts, view logic, and module responsibilities — is platform-agnostic.

---

## 1. Overview and Purpose

### 1.1 The Promise, Evidence, and Impact Triad

A data contract is the formal agreement between a data product and the consumers who depend on it. It defines three concerns that must be addressed together:

| Concern | Question | Where It Lives |
|---------|----------|----------------|
| **Promise** | What is the product committing to deliver? | Semantic module — contract definition tables |
| **Evidence** | Was the promise kept? | Observability module — validation and violation tables |
| **Impact** | Who is affected when the promise is broken? | Observability module — consumer impact tables |

Managing any one of these in isolation is insufficient. A product that defines a contract but never validates it provides no assurance. A product that validates but never registers consumers cannot determine impact when something fails. A product that tracks violations but records no history cannot demonstrate sustained compliance to auditors or regulators.

### 1.2 A Contract Is Not Just a Document

A data contract in this architecture is not a YAML file committed to a repository, nor a PDF signed by stakeholders. **A data contract is a queryable, enforceable, evidence-backed interface.** It exists as structured data within the product's own modules, enabling:

- Automated validation on every pipeline execution
- Real-time consumer impact assessment when a rule fails
- Historical evidence of compliance across any date range
- Machine-readable export to industry-standard formats (ODCS, DPDS, DCAT, DataHub, OpenMetadata)
- Audit-ready reporting without manual reconstruction

### 1.3 Scope of This Standard

This document is a **cross-cutting design standard**. It does not introduce a seventh module. Instead, it prescribes how the six existing modules work together to implement a complete data contract capability. Every data product built to the AI-Native Data Product Architecture standard must apply this standard to all public-facing output interfaces.

---

## 2. Fundamental Principle: Cross-Cutting, Not a New Module

### 2.1 Why Not a Seventh Module?

Introducing a dedicated contract module would fragment concerns that naturally belong together. Contract definitions are metadata — they belong with the Semantic module. Contract evidence is observational data — it belongs with Observability. Contract governance records are decisions and approvals — they belong with Memory. Splitting these across a new module would:

- Duplicate the role of existing modules
- Create inconsistency in where designers look for metadata
- Break the principle that each module has a single, clear responsibility
- Require new integration patterns where existing ones already work

Instead, contract concerns are assigned to the module that already owns the corresponding data concern.

### 2.2 Module Responsibility Mapping

| Contract Concern | Module | Rationale |
|-----------------|--------|-----------|
| Contract definition — what is promised | **Semantic** | Semantic owns schema metadata, naming standards, and the business meaning of entities and attributes |
| Contract versioning — history of promises | **Semantic** | Version history is metadata about the schema; it belongs alongside entity and column metadata |
| Interface specification — which views are contracted | **Semantic** | Interfaces are schema artefacts; contracted views are registered alongside entity metadata |
| Field schema — contracted types, nullability, classification | **Semantic** | Field definitions are column metadata with contract-level commitments applied |
| Quality and SLA rules — the validation criteria | **Semantic** | Rules are metadata that define the expected behaviour of data — the promise in concrete terms |
| Consumer registration — who has subscribed | **Semantic** | Consumer registration is metadata about the contract relationship |
| Validation runs — did the rules execute | **Observability** | Validation is an event — it happened at a point in time, producing metrics |
| Validation results — which rules passed or failed | **Observability** | Pass/fail is observational evidence — the proof that the promise was or was not met |
| Violations — breaches and their status | **Observability** | Violations are events with severity, duration, and remediation tracking |
| SLA status — freshness and timeliness | **Observability** | SLA status is a metric derived from execution events |
| Change log — human-readable history | **Memory** | Change history is design knowledge — it records decisions, context, and rationale |
| Design decisions — architectural choices | **Memory** | The Memory module's Documentation Sub-Module owns all architectural decision records |
| Consumer sign-off — approval for breaking changes | **Memory** | Sign-off is a governance decision — it is an approval record, not an event or metric |
| Data exposure — contracted views and access | **Domain** | The Domain module exposes the physical data; contracted interfaces are views over Domain tables |
| Access control — roles and grants | **Access** | The Access layer governs who may consume which interface at which classification level |

---

## 3. Mandatory Contract Rule

### 3.1 Every Public-Facing Interface Must Have a Contract

Any interface exposed outside the data product's internal processing boundary **must** have a registered contract before it is deployed to production. There are no exceptions to this rule beyond the formal exception process described in Section 3.3.

### 3.2 Defining "Public-Facing"

A public-facing interface is any of the following:

| Interface Type | Examples |
|----------------|---------|
| **Domain views** | Views in `{Product}_Domain_V` database consumed by external applications |
| **Access layer views** | Views in `{Product}_Access_V` database issued with grants to external roles |
| **Prediction outputs** | Feature sets, model scores, or recommendation tables consumed by downstream systems |
| **Search endpoints** | Embedding stores or search result tables queried by external agents or applications |
| **Analytical exports** | Views designed for BI tool consumption, reporting, or regulatory submissions |
| **API-backed interfaces** | Any view or table backing an API endpoint exposed outside the product boundary |

A table or view is **not** public-facing if it is:
- An internal staging or processing table used only within the product's own pipelines
- A module-internal management table (e.g., `Semantic.entity_metadata` itself)
- A temporary or work table used within a single job run

### 3.3 Exception Process

Where a public-facing interface cannot be contracted at deployment time — for example, during a time-boxed prototype or emergency break-fix deployment — the following steps are required:

1. Create a `contract_design_decision` record in the Memory module documenting:
   - The interface that is uncontracted
   - The business reason for the exception
   - The agreed deadline for contract registration
   - The approver name and approval date
2. Tag the exception with `decision_category = 'CONTRACT_EXCEPTION'`
3. Register the contract no later than the agreed deadline

An uncontracted public-facing interface without a Memory design decision record is a compliance defect.

---

## 4. Contract Table Design

This section describes the purpose and key columns for each contract table. Full DDL is provided in the deployment artefacts; this standard defines the minimum schema contract each table must satisfy.

### 4.1 Semantic Module — Contract Definition

#### 4.1.1 `contract`

Top-level contract identity record. One row per contract.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_id` | INTEGER IDENTITY | Surrogate key |
| `contract_key` | VARCHAR(100) NOT NULL UNIQUE | Human-readable identifier (e.g., `customer_current_v1`) |
| `contract_name` | VARCHAR(200) NOT NULL | Display name |
| `contract_description` | VARCHAR(2000) | Business description of what is promised |
| `product_name` | VARCHAR(128) NOT NULL | Data product this contract belongs to |
| `owner_name` | VARCHAR(200) NOT NULL | Accountable owner — person or team name |
| `owner_contact` | VARCHAR(200) | Email or channel for contract enquiries |
| `contract_status` | VARCHAR(20) NOT NULL | `DRAFT`, `ACTIVE`, `DEPRECATED`, `RETIRED` |
| `is_active` | BYTEINT NOT NULL DEFAULT 1 | Soft-delete indicator |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |
| `updated_at` | TIMESTAMP(6) WITH TIME ZONE | Record last-update timestamp |

#### 4.1.2 `contract_version`

Version history for a contract. One row per version per contract. Supports semantic versioning.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_version_id` | INTEGER IDENTITY | Surrogate key |
| `contract_id` | INTEGER NOT NULL | FK → `contract.contract_id` |
| `version_number` | VARCHAR(20) NOT NULL | Semantic version string (e.g., `2.1.0`) |
| `version_major` | SMALLINT NOT NULL | Major version component |
| `version_minor` | SMALLINT NOT NULL | Minor version component |
| `version_patch` | SMALLINT NOT NULL | Patch version component |
| `compatibility_level` | VARCHAR(20) NOT NULL | `BREAKING`, `NON_BREAKING`, `PATCH` |
| `effective_from_dt` | DATE NOT NULL | Date from which this version is binding |
| `effective_to_dt` | DATE | Date on which this version is superseded; NULL if current |
| `deprecation_notice_dt` | DATE | Date deprecation was announced to consumers |
| `retirement_dt` | DATE | Planned or actual retirement date |
| `version_status` | VARCHAR(20) NOT NULL | `DRAFT`, `ACTIVE`, `DEPRECATED`, `RETIRED` |
| `change_summary` | VARCHAR(2000) | Human-readable summary of changes in this version |
| `created_by` | VARCHAR(200) NOT NULL | Person or system that created this version |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

#### 4.1.3 `contract_interface`

The contracted view, table, or endpoint. One row per interface per contract version.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_interface_id` | INTEGER IDENTITY | Surrogate key |
| `contract_version_id` | INTEGER NOT NULL | FK → `contract_version.contract_version_id` |
| `interface_name` | VARCHAR(200) NOT NULL | Logical name of the interface (e.g., `customer_current`) |
| `interface_type` | VARCHAR(50) NOT NULL | `VIEW`, `TABLE`, `FEATURE_SET`, `PREDICTION_OUTPUT`, `SEARCH_ENDPOINT` |
| `database_name` | VARCHAR(128) NOT NULL | Physical Teradata database containing the object |
| `object_name` | VARCHAR(128) NOT NULL | Physical view or table name |
| `refresh_frequency` | VARCHAR(50) | How often data is refreshed (e.g., `15MIN`, `DAILY`, `NEAR_REAL_TIME`) |
| `freshness_sla_minutes` | INTEGER | Maximum acceptable age of data in minutes |
| `availability_sla_pct` | DECIMAL(5,2) | Minimum availability percentage (e.g., `99.5`) |
| `validation_status` | VARCHAR(20) NOT NULL DEFAULT 'UNKNOWN' | `PASSING`, `FAILING`, `WARNING`, `UNKNOWN` — updated by last validation run |
| `last_validated_dts` | TIMESTAMP(6) WITH TIME ZONE | Timestamp of last validation run |
| `is_active` | BYTEINT NOT NULL DEFAULT 1 | Soft-delete indicator |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

#### 4.1.4 `contract_field`

Field-level schema contract. One row per field per interface.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_field_id` | INTEGER IDENTITY | Surrogate key |
| `contract_interface_id` | INTEGER NOT NULL | FK → `contract_interface.contract_interface_id` |
| `field_name` | VARCHAR(128) NOT NULL | Column name in the physical object |
| `field_description` | VARCHAR(1000) | Business meaning of the field |
| `data_type` | VARCHAR(100) NOT NULL | Contracted data type (e.g., `VARCHAR(100)`, `DECIMAL(18,2)`) |
| `is_nullable` | BYTEINT NOT NULL DEFAULT 1 | 1 = nullable, 0 = NOT NULL |
| `data_classification` | VARCHAR(50) NOT NULL | `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED` |
| `is_pii` | BYTEINT NOT NULL DEFAULT 0 | 1 = contains PII |
| `is_sensitive` | BYTEINT NOT NULL DEFAULT 0 | 1 = sensitive (financial, health, etc.) |
| `breaking_change_rule` | VARCHAR(20) NOT NULL DEFAULT 'EVALUATE' | `NEVER_REMOVE`, `VERSION_BEFORE_CHANGE`, `EVALUATE` — governs what changes require MAJOR bump |
| `allowed_values_json` | JSON | Allowed-value domain, if constrained |
| `is_active` | BYTEINT NOT NULL DEFAULT 1 | Soft-delete indicator |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

#### 4.1.5 `contract_rule`

Validation rules that operationalise the contract. One row per rule per interface.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_rule_id` | INTEGER IDENTITY | Surrogate key |
| `contract_interface_id` | INTEGER NOT NULL | FK → `contract_interface.contract_interface_id` |
| `contract_field_id` | INTEGER | FK → `contract_field.contract_field_id`; NULL for table-level rules |
| `rule_name` | VARCHAR(200) NOT NULL | Unique name for this rule (e.g., `customer_id_not_null`) |
| `rule_type` | VARCHAR(50) NOT NULL | `QUALITY`, `FRESHNESS`, `SLA`, `REFERENTIAL`, `VALUE_DOMAIN`, `ROW_COUNT` |
| `rule_description` | VARCHAR(1000) NOT NULL | What the rule checks and why it matters |
| `rule_severity` | VARCHAR(20) NOT NULL | `CRITICAL`, `HIGH`, `MEDIUM`, `LOW` — see Section 8 |
| `rule_logic` | VARCHAR(4000) | SQL expression or prose description of the validation check |
| `threshold_value` | DECIMAL(15,4) | Numeric threshold (e.g., completeness >= 0.99) |
| `threshold_operator` | VARCHAR(10) | `>=`, `<=`, `=`, `>`, `<` |
| `is_enforced` | BYTEINT NOT NULL DEFAULT 1 | 1 = failures generate violations; 0 = monitoring only |
| `is_active` | BYTEINT NOT NULL DEFAULT 1 | Soft-delete indicator |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

#### 4.1.6 `contract_consumer`

Registered consumers of a contracted interface. One row per consumer per contract version.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_consumer_id` | INTEGER IDENTITY | Surrogate key |
| `contract_version_id` | INTEGER NOT NULL | FK → `contract_version.contract_version_id` — the version this consumer is registered against |
| `contract_interface_id` | INTEGER NOT NULL | FK → `contract_interface.contract_interface_id` |
| `consumer_name` | VARCHAR(200) NOT NULL | Name of the consuming system, team, or application |
| `consumer_owner` | VARCHAR(200) NOT NULL | Accountable owner of the consumer |
| `consumer_contact` | VARCHAR(200) | Email or channel for consumer notifications |
| `consumer_purpose` | VARCHAR(1000) NOT NULL | Why this consumer uses the interface (business purpose) |
| `data_classification_access` | VARCHAR(50) NOT NULL | Highest classification level the consumer is permitted to access |
| `notification_channel` | VARCHAR(200) | Where to route contract violation notifications (e.g., Slack channel, email group) |
| `registration_status` | VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' | `ACTIVE`, `MIGRATING`, `DEREGISTERED` |
| `signoff_status` | VARCHAR(20) NOT NULL DEFAULT 'NOT_REQUIRED' | `NOT_REQUIRED`, `PENDING`, `APPROVED`, `REJECTED` |
| `registered_dt` | DATE NOT NULL | Date consumer was registered |
| `deregistered_dt` | DATE | Date consumer was removed from this version |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

---

### 4.2 Observability Module — Contract Evidence

#### 4.2.1 `contract_validation_run`

Execution record for each validation cycle. One row per validation run per interface.

| Column | Type | Purpose |
|--------|------|---------|
| `validation_run_id` | INTEGER IDENTITY | Surrogate key |
| `contract_interface_id` | INTEGER NOT NULL | FK → `contract_interface.contract_interface_id` |
| `contract_version_id` | INTEGER NOT NULL | FK → `contract_version.contract_version_id` — version validated |
| `run_dts` | TIMESTAMP(6) WITH TIME ZONE NOT NULL | When validation was executed |
| `run_status` | VARCHAR(20) NOT NULL | `PASS`, `FAIL`, `WARN`, `ERROR` — overall result for this run |
| `rules_evaluated` | INTEGER | Total number of rules evaluated |
| `rules_passed` | INTEGER | Number of rules that passed |
| `rules_failed` | INTEGER | Number of rules that failed |
| `rules_warned` | INTEGER | Number of rules in WARNING state |
| `triggered_by` | VARCHAR(100) | Process or agent that triggered validation (e.g., `ETL_NIGHTLY`, `AGENT`) |
| `batch_key` | VARCHAR(100) | Links to `change_event.batch_key` for correlation with ETL cycles |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

#### 4.2.2 `contract_validation_result`

Rule-level pass/fail evidence. One row per rule per validation run.

| Column | Type | Purpose |
|--------|------|---------|
| `validation_result_id` | INTEGER IDENTITY | Surrogate key |
| `validation_run_id` | INTEGER NOT NULL | FK → `contract_validation_run.validation_run_id` |
| `contract_rule_id` | INTEGER NOT NULL | FK → `contract_rule.contract_rule_id` |
| `result_status` | VARCHAR(20) NOT NULL | `PASS`, `FAIL`, `WARN`, `SKIP` |
| `measured_value` | DECIMAL(15,6) | Actual measured value (e.g., completeness score 0.9987) |
| `threshold_value` | DECIMAL(15,6) | Threshold value at time of evaluation |
| `records_evaluated` | INTEGER | Number of records examined |
| `records_failing` | INTEGER | Number of records that failed the rule |
| `result_detail` | VARCHAR(2000) | Human-readable explanation of the result |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

#### 4.2.3 `contract_violation`

Violation history. One row per violation event. Violations are created when a rule fails and `is_enforced = 1`.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_violation_id` | INTEGER IDENTITY | Surrogate key |
| `validation_result_id` | INTEGER NOT NULL | FK → `contract_validation_result.validation_result_id` — the result that triggered this violation |
| `contract_rule_id` | INTEGER NOT NULL | FK → `contract_rule.contract_rule_id` |
| `contract_interface_id` | INTEGER NOT NULL | FK → `contract_interface.contract_interface_id` |
| `violation_type` | VARCHAR(50) NOT NULL | See Section 8.2 for standard violation event types |
| `violation_severity` | VARCHAR(20) NOT NULL | `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFO` |
| `violation_dts` | TIMESTAMP(6) WITH TIME ZONE NOT NULL | When the violation occurred |
| `violation_detail` | VARCHAR(2000) | Description of what failed and measured vs. expected values |
| `consumer_impact` | VARCHAR(20) NOT NULL DEFAULT 'UNKNOWN' | `NONE`, `LOW`, `HIGH`, `CRITICAL` |
| `affected_consumer_count` | INTEGER DEFAULT 0 | Number of registered consumers potentially affected |
| `remediation_status` | VARCHAR(20) NOT NULL DEFAULT 'OPEN' | `OPEN`, `IN_PROGRESS`, `RESOLVED`, `ACCEPTED` |
| `remediation_notes` | VARCHAR(2000) | Notes from the team resolving the violation |
| `resolved_dts` | TIMESTAMP(6) WITH TIME ZONE | When remediation was completed |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

#### 4.2.4 `contract_sla_status`

Current SLA status for each contracted interface. One row per interface, upserted on each validation cycle.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_sla_status_id` | INTEGER IDENTITY | Surrogate key |
| `contract_interface_id` | INTEGER NOT NULL | FK → `contract_interface.contract_interface_id` |
| `freshness_status` | VARCHAR(20) NOT NULL | `FRESH`, `STALE`, `CRITICAL` — based on `freshness_sla_minutes` |
| `last_successful_refresh_dts` | TIMESTAMP(6) WITH TIME ZONE | Most recent successful pipeline run for this interface |
| `minutes_since_refresh` | INTEGER | Derived: minutes since `last_successful_refresh_dts` |
| `timeliness_status` | VARCHAR(20) NOT NULL | `ON_TIME`, `DELAYED`, `BREACHED` — based on scheduled vs. actual refresh |
| `availability_status` | VARCHAR(20) NOT NULL | `AVAILABLE`, `DEGRADED`, `UNAVAILABLE` |
| `sla_status_dts` | TIMESTAMP(6) WITH TIME ZONE NOT NULL | Timestamp when this status was last computed |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

---

### 4.3 Memory Module — Contract Governance

#### 4.3.1 `contract_change_log`

Human-readable history and rationale for all contract changes. One row per change event.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_change_log_id` | INTEGER IDENTITY | Surrogate key |
| `contract_id` | INTEGER NOT NULL | FK → `contract.contract_id` |
| `contract_version_id` | INTEGER | FK → `contract_version.contract_version_id`; NULL for pre-version changes |
| `change_type` | VARCHAR(50) NOT NULL | `VERSION_CREATED`, `RULE_ADDED`, `RULE_CHANGED`, `FIELD_ADDED`, `FIELD_DEPRECATED`, `CONSUMER_REGISTERED`, `CONSUMER_DEREGISTERED`, `STATUS_CHANGED` |
| `change_summary` | VARCHAR(2000) NOT NULL | Human-readable description of what changed |
| `change_rationale` | VARCHAR(2000) | Why the change was made |
| `changed_by` | VARCHAR(200) NOT NULL | Person or system that made the change |
| `change_dts` | TIMESTAMP(6) WITH TIME ZONE NOT NULL | When the change was made |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

#### 4.3.2 `contract_design_decision`

Architectural decisions affecting contracts. Extends the Memory module's `Design_Decision` pattern for contract-specific concerns.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_decision_id` | INTEGER IDENTITY | Surrogate key |
| `contract_id` | INTEGER | FK → `contract.contract_id`; NULL for cross-contract decisions |
| `decision_id` | VARCHAR(50) NOT NULL | Decision identifier (e.g., `DD-CONTRACT-001`) |
| `decision_title` | VARCHAR(200) NOT NULL | Short title for the decision |
| `decision_context` | VARCHAR(4000) NOT NULL | The situation or problem that prompted this decision |
| `decision_made` | VARCHAR(4000) NOT NULL | What was decided |
| `decision_rationale` | VARCHAR(4000) NOT NULL | Why this decision was made over alternatives |
| `decision_category` | VARCHAR(50) NOT NULL | `ARCHITECTURE`, `VERSIONING`, `CONSUMER`, `SLA`, `CONTRACT_EXCEPTION`, `INTEGRATION` |
| `decision_status` | VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' | `ACTIVE`, `SUPERSEDED`, `DEPRECATED` |
| `decided_by` | VARCHAR(200) NOT NULL | Person who made or approved the decision |
| `decided_dt` | DATE NOT NULL | Date the decision was made |
| `review_dt` | DATE | Next scheduled review date |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

#### 4.3.3 `contract_consumer_signoff`

Evidence of consumer acknowledgement for breaking changes. One row per sign-off event.

| Column | Type | Purpose |
|--------|------|---------|
| `contract_consumer_signoff_id` | INTEGER IDENTITY | Surrogate key |
| `contract_consumer_id` | INTEGER NOT NULL | FK → `contract_consumer.contract_consumer_id` |
| `contract_version_id` | INTEGER NOT NULL | FK → `contract_version.contract_version_id` — the new MAJOR version requiring sign-off |
| `signoff_status` | VARCHAR(20) NOT NULL | `PENDING`, `APPROVED`, `REJECTED`, `ESCALATED` |
| `signoff_dts` | TIMESTAMP(6) WITH TIME ZONE | When sign-off was recorded |
| `signed_off_by` | VARCHAR(200) | Name of person who provided sign-off |
| `signoff_notes` | VARCHAR(2000) | Notes from the consumer (e.g., migration timeline, conditions) |
| `requested_dts` | TIMESTAMP(6) WITH TIME ZONE NOT NULL | When sign-off was requested |
| `deadline_dt` | DATE NOT NULL | Date by which sign-off must be received |
| `created_at` | TIMESTAMP(6) WITH TIME ZONE | Record creation timestamp |

---

## 5. Contract Lifecycle

The contract lifecycle has eight steps. Each step has defined entry criteria, responsible parties, and output destinations.

### 5.1 Lifecycle Overview

```
DEFINE → VERSION → REGISTER → VALIDATE → ENFORCE → ALERT → EVIDENCE → CHANGE-CONTROL
```

### 5.2 Step-by-Step Description

#### Step 1: DEFINE

**What happens:** The data product designer identifies all public-facing interfaces and writes the contract definition — what the interface provides, what fields are exposed, what quality rules apply, and what SLAs are committed to.

**Responsible:** Data product designer.

**Output:** `contract`, `contract_interface`, `contract_field`, `contract_rule` rows in `{Product}_Semantic`. `contract_status = 'DRAFT'`.

**Entry criteria:** Interface design is finalised and the view or table DDL is written (or at least specified).

**Exit criteria:** All fields documented in `contract_field`. At least one rule per field with `breaking_change_rule` set. Owner identified. Contract status is `DRAFT`.

---

#### Step 2: VERSION

**What happens:** The designer assigns a semantic version to the contract definition, records the effective date, and classifies the compatibility level. For initial releases this is always `1.0.0`.

**Responsible:** Data product designer.

**Output:** `contract_version` row in `{Product}_Semantic`.

**Entry criteria:** Contract definition is complete and in `DRAFT` status.

**Exit criteria:** `contract_version` row exists with `version_status = 'DRAFT'`, `effective_from_dt` set, `compatibility_level` assigned.

---

#### Step 3: REGISTER

**What happens:** The contract is registered in the Semantic module, making it discoverable to agents and consumers. Known consumers are registered in `contract_consumer` at this step. The contract version moves to `ACTIVE`.

**Responsible:** Data product designer and consuming team owners.

**Output:** `contract_version.version_status = 'ACTIVE'`, `contract_status = 'ACTIVE'`, `contract_consumer` rows for all known consumers.

**Entry criteria:** Interface has been deployed to the `{Product}_Domain_V` or `{Product}_Access_V` database.

**Exit criteria:** Contract is `ACTIVE`. All known consumers registered. `contract_interface.validation_status = 'UNKNOWN'` (not yet validated).

---

#### Step 4: VALIDATE

**What happens:** The validation pipeline executes the rules defined in `contract_rule` against the contracted interface. This should be triggered at least once per pipeline refresh cycle. Results are written to `contract_validation_run` and `contract_validation_result`.

**Responsible:** ETL pipeline or validation agent (automated).

**Output:** `contract_validation_run` and `contract_validation_result` rows in `{Product}_Observability`. `contract_interface.validation_status` is updated to `PASSING`, `FAILING`, or `WARNING`.

**Entry criteria:** Contract is `ACTIVE`. Interface data is available.

**Exit criteria:** A `contract_validation_run` row exists. All enforced rules have a `contract_validation_result` row. `contract_interface.last_validated_dts` is updated.

---

#### Step 5: ENFORCE

**What happens:** Where a rule has `is_enforced = 1` and the validation result is `FAIL`, a `contract_violation` record is created automatically. Enforcement transforms a monitoring failure into a tracked incident.

**Responsible:** Validation pipeline (automated).

**Output:** `contract_violation` rows in `{Product}_Observability` with `remediation_status = 'OPEN'`. Consumer impact is assessed: `affected_consumer_count` is populated by joining `contract_consumer`.

**Entry criteria:** A `FAIL` result exists in `contract_validation_result` for a rule with `is_enforced = 1`.

**Exit criteria:** `contract_violation` row created. `consumer_impact` assessed. `affected_consumer_count` populated.

---

#### Step 6: ALERT

**What happens:** Open violations trigger notifications to the registered consumers and the contract owner via the notification channels recorded in `contract_consumer.notification_channel`. Routing follows the severity rules in Section 8.

**Responsible:** Alerting integration (automated) — typically a procedure or scheduled agent that reads open violations and dispatches notifications.

**Output:** Notifications delivered to PagerDuty, ServiceNow, Slack, email, or whichever channels are configured. No new database rows are created at this step; this is a dispatch operation.

**Entry criteria:** `contract_violation.remediation_status = 'OPEN'`.

**Exit criteria:** Notification dispatched. Contract owner and affected consumers are aware of the violation.

---

#### Step 7: EVIDENCE

**What happens:** Over time, the accumulated `contract_validation_run`, `contract_validation_result`, `contract_violation`, and `contract_sla_status` rows build an evidence record. This evidence is surfaced through the view groups defined in Section 12 and the dashboards in Section 13.

**Responsible:** Ongoing — the system produces evidence continuously; the designer configures retention policies.

**Output:** Evidence views and dashboards available for consumer queries, SLA reporting, regulatory review, and audit.

**Entry criteria:** At least one completed validation cycle.

**Exit criteria:** Not terminal — evidence accumulates for the life of the contract.

---

#### Step 8: CHANGE-CONTROL

**What happens:** When the contract must change — a new field, a type change, a rule tightening, or a MAJOR breaking change — the change-control process governs how the change is introduced. See Section 6 for versioning rules and Section 7 for consumer sign-off obligations.

**Responsible:** Data product designer (change author). Contract owner (approver). Consumer owners (sign-off for MAJOR changes).

**Output:** New `contract_version` row. `contract_change_log` entry. `contract_consumer_signoff` rows for affected consumers (MAJOR changes only). Design decision record if the change is material.

**Entry criteria:** A change to the interface or its rules is required.

**Exit criteria:** New version is `ACTIVE`. Previous version is `DEPRECATED` with `deprecation_notice_dt` set and `retirement_dt` declared. All affected consumers notified and sign-off collected (MAJOR changes only).

---

## 6. Semantic Versioning and Breaking-Change Rules

### 6.1 Version Scheme

Contracts use **MAJOR.MINOR.PATCH** semantic versioning, stored as three separate integer columns (`version_major`, `version_minor`, `version_patch`) and a formatted string (`version_number`).

| Component | Increment When |
|-----------|----------------|
| **MAJOR** | A breaking change is introduced — see Section 6.2 |
| **MINOR** | A non-breaking additive change is introduced — see Section 6.3 |
| **PATCH** | A correction that does not change the interface contract (e.g., fixing a rule description, correcting a threshold that was misconfigured) |

### 6.2 Breaking Changes — Require MAJOR Increment

Any of the following changes constitutes a breaking change and must be introduced as a new MAJOR version:

| Change Type | Example |
|-------------|---------|
| **Field removal** | Removing `account_balance` from a contracted interface |
| **Type narrowing** | Changing `VARCHAR(200)` to `VARCHAR(50)` — existing consumers may have data that no longer fits |
| **Null constraint tightening** | Changing a nullable field to NOT NULL — existing consumers that write null values would break |
| **Row filter that excludes previously-included rows** | Adding a `WHERE status = 'ACTIVE'` filter to a view that previously returned all statuses |
| **Primary key or unique key change** | Changing the grain of the interface |
| **SLA shortening** | Reducing `freshness_sla_minutes` — consumers that scheduled on the previous SLA would now be in breach |
| **Data classification elevation** | Moving a field from `INTERNAL` to `RESTRICTED` — existing consumers with INTERNAL access would lose visibility |
| **Renaming a field** | Changing `cust_id` to `customer_id` — all consumer queries break |

### 6.3 Non-Breaking Changes — MINOR Increment

The following changes are additive and do not break existing consumers:

| Change Type | Example |
|-------------|---------|
| **New nullable field** | Adding `preferred_language` to an existing interface |
| **Widened type** | Changing `VARCHAR(50)` to `VARCHAR(200)` |
| **Extended allowed-value list** | Adding `'DIGITAL'` to a status code list |
| **New interface on the same contract** | Adding a second view to an existing contract version |
| **New quality rule** | Adding a monitoring rule (does not affect data shape) |
| **SLA extension** | Increasing `freshness_sla_minutes` — looser SLA, consumers are not disadvantaged |

### 6.4 Deprecation Timeline

Before a MAJOR version becomes effective, the following minimum notice periods apply:

| Consumer Tier | Minimum Notice Period |
|---------------|-----------------------|
| External/regulated consumers | 90 days |
| Internal consumers — production systems | 30 days |
| Internal consumers — analytical/reporting | 14 days |
| Development/prototype consumers | 7 days |

The notice period begins on `contract_version.deprecation_notice_dt`. The `retirement_dt` must be at least the minimum notice period after the `deprecation_notice_dt`.

### 6.5 Consumer Notification Obligations for MAJOR Changes

When a MAJOR version is created:

1. All consumers registered against the current ACTIVE version must receive a notification immediately (see Section 8 for routing).
2. A `contract_consumer_signoff` row must be created for each consumer with `signoff_status = 'PENDING'`.
3. The previous version must not be retired until either all consumers have `signoff_status = 'APPROVED'` or the `retirement_dt` has passed (at which point outstanding non-responses are escalated).
4. The change log entry must record all breaking changes and the migration path for consumers.

---

## 7. Consumer Registration and Sign-Off

### 7.1 Registration Requirement

Every consumer of a public-facing interface **must** be registered in `contract_consumer` against the specific contract version they are consuming. Consumption without registration is not permitted once a contract is `ACTIVE`.

This requirement applies to:
- Internal applications and ETL pipelines
- AI agents querying contracted interfaces
- External applications issued database grants
- BI tools and reporting systems
- Regulatory reporting processes

### 7.2 Registration Fields

| Field | Requirement | Notes |
|-------|-------------|-------|
| `consumer_name` | Mandatory | System name or team name |
| `consumer_owner` | Mandatory | Named individual accountable for the consumer |
| `consumer_contact` | Mandatory | Email or channel — must be reachable within 24 hours |
| `consumer_purpose` | Mandatory | Specific business use — not "general analytical use" |
| `data_classification_access` | Mandatory | Must match the Access layer grant level |
| `notification_channel` | Mandatory | Must be monitored by the consumer team |
| `registration_status` | Set to `ACTIVE` at registration | |

### 7.3 Sign-Off for MAJOR Version Changes

When a MAJOR version is introduced:

1. The product team creates a `contract_consumer_signoff` row for each active consumer with `signoff_status = 'PENDING'` and `deadline_dt` set per Section 6.4.
2. The consumer owner reviews the breaking changes and migration path in `contract_change_log`.
3. The consumer owner sets `signoff_status = 'APPROVED'` or `'REJECTED'` in `contract_consumer_signoff`, with `signoff_notes` documenting their migration timeline.
4. A rejection triggers a design decision record (`contract_design_decision`) documenting the resolution approach.

### 7.4 Consumer Migration When Deprecating a Version

When a version enters `DEPRECATED` status:

1. Each registered consumer is notified immediately via `notification_channel`.
2. The consumer should update their registration to the new version (update `contract_version_id` in `contract_consumer`) once migration is complete, and update `registration_status = 'DEREGISTERED'` on the old version entry.
3. Consumers still registered against a `DEPRECATED` version appear in the `contract_deprecated_consumer` governance view (Section 12.4) and are subject to escalation.
4. The product team must not retire the old version's physical objects (views, tables) until `contract_deprecated_consumer` returns zero rows for that version.

---

## 8. Violation Events and Alerting

### 8.1 Severity Levels

| Severity | Definition | Required Action | SLA for Remediation |
|----------|-----------|-----------------|---------------------|
| **CRITICAL** | A violation that causes data consumers to receive incorrect, incomplete, or unavailable data with direct business impact | Immediate response. Notify contract owner + all affected consumers. Escalate to on-call if unresolved within threshold. | 4 hours |
| **HIGH** | A violation that degrades data quality or freshness materially, likely affecting consumer decisions | Same business-day response. Notify contract owner + affected consumers. | 24 hours |
| **MEDIUM** | A violation that degrades quality below threshold but with limited immediate consumer impact | Next-business-day response. Notify contract owner. | 5 business days |
| **LOW** | A violation that fails a quality rule but where the impact is negligible or the rule is advisory | Inform team. Log for trend analysis. | 14 business days |
| **INFO** | An informational event — no rule failure, but a state change worth recording (e.g., version deprecated, consumer registered) | No remediation required. Record for audit. | N/A |

### 8.2 Standard Violation Event Types

The `contract_violation.violation_type` column must use the following standard event type codes:

| Event Type | Triggered When |
|------------|---------------|
| `CONTRACT_VALIDATION_FAILED` | A validation rule with `is_enforced = 1` produces a `FAIL` result in `contract_validation_result` |
| `CONTRACT_SLA_BREACH` | `contract_sla_status.freshness_status = 'CRITICAL'` or `timeliness_status = 'BREACHED'` |
| `CONTRACT_SCHEMA_MISMATCH` | The physical schema of the contracted object differs from the field definitions in `contract_field` (detected by a schema-comparison validation rule) |
| `CONTRACT_CONSUMER_IMPACT` | A violation has `affected_consumer_count > 0` — this event type co-exists with the primary violation type to trigger consumer-specific notification routing |
| `CONTRACT_VERSION_DEPRECATED` | A contract version transitions to `DEPRECATED` status — recorded as an INFO-severity violation for audit purposes |
| `CONTRACT_BREAKING_CHANGE_PENDING` | A new MAJOR version is in `DRAFT` status with `effective_from_dt` in the future — recorded to notify consumers of an upcoming breaking change |

### 8.3 Notification Routing

Violations route to the organisation's standard operational channels via the Observability module's alerting integration. The routing matrix below defines the minimum notification targets by severity:

| Severity | Contract Owner | Affected Consumers | PagerDuty / On-Call | ServiceNow Ticket | Slack / Email |
|----------|---------------|-------------------|----------------------|-------------------|---------------|
| CRITICAL | ✅ Immediate | ✅ Immediate | ✅ | ✅ | ✅ |
| HIGH | ✅ Immediate | ✅ Immediate | ❌ | ✅ | ✅ |
| MEDIUM | ✅ Same day | ❌ | ❌ | ✅ | ✅ |
| LOW | ✅ Weekly digest | ❌ | ❌ | ❌ | ✅ |
| INFO | ❌ | ❌ | ❌ | ❌ | ✅ Audit log only |

The actual dispatch mechanism — whether via a stored procedure, a scheduled agent, a webhook, or an external monitoring tool — is an implementation concern. The standard requires only that `contract_violation` and `contract_consumer_impact` views provide the routing context required by the dispatch mechanism (consumer, channel, severity).

---

## 9. Access, Roles, and Grants Tied to Contracts

### 9.1 Interface-Level Grants

The Access layer must issue grants at the **interface level** — against the contracted view or table — not directly against the underlying Domain tables. This principle ensures:

- Access can be revoked cleanly when a contract version is deprecated
- The contract version governs which data shape the consumer receives
- Field-level classification (`contract_field.data_classification`) drives what fields appear in the interface view

#### Pattern

```sql
-- Grant access to the contracted interface view, not the raw Domain table
GRANT SELECT ON {Product}_Access_V.customer_current_v1 TO customer_ops_read_role;

-- Do NOT grant directly to the Domain table
-- GRANT SELECT ON {Product}_Domain.Party_H TO customer_ops_read_role;  -- NOT PERMITTED
```

### 9.2 Role Naming Convention

Roles tied to contracted interfaces follow this naming pattern:

```
{product}_{interface}_{accesslevel}_role
```

| Component | Values | Example |
|-----------|--------|---------|
| `{product}` | Abbreviated product name | `custmgmt` |
| `{interface}` | Interface name from `contract_interface.interface_name` | `customer_current` |
| `{accesslevel}` | `read`, `write`, `admin` | `read` |

**Examples:**

| Role Name | Interface | Access Level |
|-----------|-----------|--------------|
| `custmgmt_customer_current_read_role` | `customer_current` | Read-only |
| `acctmgmt_account_balance_current_read_role` | `account_balance_current` | Read-only |
| `frauddet_fraud_near_real_time_read_role` | `fraud_near_real_time` | Read-only |

### 9.3 Role Lifecycle Tied to Contract Version

When a contract version is deprecated:

1. The corresponding Access layer role must be reviewed for retirement.
2. Before retiring the role, confirm no consumers still depend on it by checking `contract_deprecated_consumer`.
3. Once all consumers have migrated to the new version's role, the deprecated-version role may be revoked.

### 9.4 Field-Level Access Governed by Classification

Fields in `contract_field` carry a `data_classification`. Views in the Access layer should filter or mask fields based on the consuming role's permitted classification level:

| Classification | Visibility to `read` role |
|----------------|--------------------------|
| `PUBLIC` | Always visible |
| `INTERNAL` | Visible with internal access role |
| `CONFIDENTIAL` | Visible only with confidential access role; otherwise masked or excluded |
| `RESTRICTED` | Visible only with restricted access role; otherwise excluded from view |

Masking or exclusion is implemented in the view definition in the `{Product}_Access_V` database. The contract field classification is the authoritative source for what masking to apply.

---

## 10. External Standards Integration

The contract backbone is designed to export and import from industry-standard formats. This section defines which contract tables feed each standard, the direction of integration, and any special handling required.

### 10.1 ODCS — Open Data Contract Standard

**What it is:** ODCS (https://bitol.io/open-data-contract-standard/) is the primary open standard for expressing data contracts in a YAML format, with fields for schema, quality rules, SLAs, and consumer terms.

**Direction:** Outbound (export from Semantic tables to ODCS YAML).

**Source tables:**

| ODCS Section | Source Table | Key Fields |
|-------------|--------------|------------|
| `info` (contract identity) | `contract`, `contract_version` | `contract_key`, `contract_name`, `owner_name`, `version_number` |
| `schema` (field definitions) | `contract_interface`, `contract_field` | `object_name`, `field_name`, `data_type`, `is_nullable`, `data_classification` |
| `quality` (validation rules) | `contract_rule` | `rule_name`, `rule_type`, `rule_logic`, `threshold_value`, `rule_severity` |
| `sla` (service levels) | `contract_interface` | `freshness_sla_minutes`, `availability_sla_pct` |
| `consumers` | `contract_consumer` | `consumer_name`, `consumer_purpose`, `data_classification_access` |

**Implementation:** Deploy the `contract_odcs_export_v` view (see Section 12.5) to produce a row-per-contract JSON or YAML structure consumable by ODCS tooling.

**Repository artefact:** `/contracts/{interface_name}.odcs.yaml` — see Section 11.

---

### 10.2 DPDS / OpenDataMesh — Data Product Descriptor Specification

**What it is:** DPDS (https://dpds.opendatamesh.org/) describes a data product's ports — its input and output interfaces — in a machine-readable descriptor. It aligns with the Data Mesh architectural model.

**Direction:** Outbound (export from `contract_interface` to DPDS port descriptors).

**Source tables:**

| DPDS Section | Source Table | Key Fields |
|-------------|--------------|------------|
| `outputPorts` | `contract_interface` | `interface_name`, `interface_type`, `database_name`, `object_name`, `refresh_frequency` |
| `promises` | `contract_version` | `version_number`, `compatibility_level`, `effective_from_dt` |
| `contracts` | `contract`, `contract_rule` | Quality and SLA promises per port |

**Implementation:** Deploy the `contract_dpds_port_export_v` view to shape each contracted interface as a DPDS output port descriptor.

**Repository artefact:** `/dpds/product.dpds.yaml`.

---

### 10.3 ODPS — Open Data Product Specification

**What it is:** ODPS (https://opendataproducts.org/) describes a data product's full profile: business description, licensing, pricing, data quality, and access terms.

**Direction:** Outbound (export from `contract` + product metadata to ODPS YAML).

**Source tables:**

| ODPS Section | Source Table | Key Fields |
|-------------|--------------|------------|
| `product` (identity) | `contract` | `product_name`, `owner_name`, `contract_description` |
| `dataQuality` | `contract_rule`, `contract_validation_run` | Rule definitions and latest pass rates |
| `dataAccess` | `contract_consumer`, `contract_interface` | Access terms, classification, consumer registration |

**Repository artefact:** `/odps/product-profile.yaml`.

---

### 10.4 DCAT — Data Catalogue Vocabulary

**What it is:** DCAT (https://www.w3.org/TR/vocab-dcat/) is a W3C standard for publishing data catalogue metadata in RDF/JSON-LD, enabling interoperability with open data portals and enterprise catalogues.

**Direction:** Outbound (export from `contract` + `contract_version` to DCAT Dataset descriptors).

**Source tables:**

| DCAT Property | Source Table | Key Fields |
|--------------|--------------|------------|
| `dcat:Dataset` | `contract`, `contract_version` | `contract_name`, `contract_description`, `effective_from_dt` |
| `dcat:Distribution` | `contract_interface` | `object_name`, `database_name`, `interface_type` |
| `dcat:keyword` | `contract_field` | `data_classification` for tagging |
| `dct:publisher` | `contract` | `owner_name`, `owner_contact` |

**Implementation:** Deploy `contract_dcat_dataset_export_v` to produce JSON-LD output for publication to open data portals or DCAT-compatible catalogues.

**Repository artefact:** `/catalog/dcat-export.jsonld`.

---

### 10.5 DataHub

**What it is:** DataHub (https://datahubproject.io/) is an open-source metadata platform with native data contract assertion support.

**Direction:** Bidirectional. Outbound for publishing contract definitions and assertions. Inbound for pulling consumer registration and stewardship updates made in the DataHub UI.

**Outbound (Teradata → DataHub):**

| DataHub Concept | Source Table |
|-----------------|-------------|
| Dataset (the contracted interface) | `contract_interface` |
| Assertions (quality rules) | `contract_rule` |
| Assertion run results | `contract_validation_result` |
| Contracts | `contract`, `contract_version` |
| Owners / consumers | `contract`, `contract_consumer` |

**Inbound (DataHub → Teradata):**

| DataHub Action | Target Table |
|---------------|-------------|
| Consumer registration via DataHub subscription | `contract_consumer` |
| Assertion status updates | `contract_interface.validation_status` |

**Repository artefact:** `/catalog/datahub-publish.yaml`.

---

### 10.6 OpenMetadata

**What it is:** OpenMetadata (https://open-metadata.org/) is an open-source metadata and data governance platform with native support for data quality tests.

**Direction:** Bidirectional. Outbound for quality test definitions and results. Inbound for stewardship updates.

**Outbound (Teradata → OpenMetadata):**

| OpenMetadata Concept | Source Table |
|---------------------|-------------|
| Table / Column metadata | `contract_interface`, `contract_field` |
| Data Quality Tests | `contract_rule` |
| Test Results | `contract_validation_result` |

**Inbound (OpenMetadata → Teradata):**

| OpenMetadata Action | Target Table |
|--------------------|-------------|
| Quality test result updates | `contract_validation_result`, `contract_interface.validation_status` |

**Repository artefact:** `/catalog/openmetadata-publish.yaml`.

---

### 10.7 OpenLineage

**What it is:** OpenLineage (https://openlineage.io/) is an open framework for data lineage collection and analysis. Foundational lineage support already exists in the Observability Module Design Standard via the `data_lineage` and `lineage_run` tables, with OpenLineage-aligned columns (`openlineage_job_name`, `openlineage_namespace`, `openlineage_run_id`).

**Direction:** Outbound extension — the contract standard adds contract context facets to existing OpenLineage events.

**Contract extension:** When emitting OpenLineage RunEvent payloads from `lineage_run`, include the following custom facets to carry contract context:

| Custom Facet | Source | Purpose |
|-------------|--------|---------|
| `contract_id` | `contract_interface.contract_id` (via join) | Links the lineage run to the contract governing the target interface |
| `contract_version` | `contract_version.version_number` | Records which contract version was in force during this run |
| `validation_run_id` | `contract_validation_run.validation_run_id` | Correlates the lineage run with its contract validation cycle |
| `validation_status` | `contract_validation_run.run_status` | Surface contract pass/fail alongside lineage execution status |

**Source tables used:** `data_lineage`, `lineage_run`, `contract_interface`, `contract_version`, `contract_validation_run`.

**Implementation:** Extend the OpenLineage RunEvent generation query in the Observability Module Design Standard (Section 5.1.3) to include contract facets via a join to `contract_interface` on `target_database` + `target_table`.

**Repository artefact:** `/lineage/openlineage-mappings.yaml` — documents the facet schema and join conditions.

---

### 10.8 OpenTelemetry

**What it is:** OpenTelemetry (https://opentelemetry.io/) provides a vendor-neutral observability framework for traces, metrics, and logs. Foundational telemetry support exists in this architecture for pipeline and agent observability.

**Direction:** Outbound extension — the contract standard adds standard contract attributes to existing OpenTelemetry spans and metrics.

**Standard Contract Attributes:**

The following attributes must be added to any OpenTelemetry span or metric that relates to a data pipeline touching a contracted interface:

| Attribute Name | Type | Source | Example |
|----------------|------|--------|---------|
| `data_product.contract.id` | string | `contract.contract_key` | `customer_current_v2` |
| `data_product.contract.version` | string | `contract_version.version_number` | `2.1.0` |
| `data_product.contract.rule_id` | string | `contract_rule.rule_name` | `customer_id_not_null` |
| `data_product.contract.rule_severity` | string | `contract_rule.rule_severity` | `CRITICAL` |
| `data_product.contract.validation_status` | string | `contract_validation_run.run_status` | `PASS` |

**Repository artefact:** `/telemetry/opentelemetry-attributes.yaml` — documents the full attribute schema with descriptions and example values.

---

## 11. Repository and File Structure

Every data product repository should include contract artefacts alongside the product's Teradata DDL. The following layout is the recommended standard structure:

```
/product-repo
  /contracts
    customer_current.odcs.yaml          -- ODCS contract artefact for customer_current interface
    account_balance.odcs.yaml           -- ODCS contract artefact for account_balance interface
  /dpds
    product.dpds.yaml                   -- DPDS port descriptor for all interfaces
  /odps
    product-profile.yaml                -- ODPS product profile
  /teradata
    domain/                             -- Domain module DDL (tables, views)
    semantic/                           -- Semantic module DDL (metadata + contract definition tables)
    observability/                      -- Observability module DDL (validation + violation tables)
    memory/                             -- Memory module DDL (change log + decision tables)
  /lineage
    openlineage-mappings.yaml           -- OpenLineage facet schema and join conditions
  /telemetry
    opentelemetry-attributes.yaml       -- OpenTelemetry contract attribute definitions
  /catalog
    dcat-export.jsonld                  -- DCAT dataset descriptor for open data publication
    datahub-publish.yaml                -- DataHub assertion and contract publishing config
    openmetadata-publish.yaml           -- OpenMetadata quality test publishing config
```

**Naming conventions:**
- ODCS files are named `{interface_name}.odcs.yaml`, matching `contract_interface.interface_name`
- DPDS and ODPS files are product-level, not interface-level
- Catalogue files use the standard tool name as the prefix

---

## 12. Contract View Groups

The contract backbone exposes standardised view groups that act as the metadata API of the contract system. Any product built to this standard can serve the same queries and dashboards without bespoke reporting work.

Views in the Discovery and Consumer groups are deployed to the `{Product}_Semantic_V` database. Evidence views are deployed to `{Product}_Observability_V`. Governance views are deployed to `{Product}_Memory_V`.

### 12.1 Discovery Views — `{Product}_Semantic_V`

**Purpose:** Enable agents and consumers to find active contracts, interfaces, fields, and rules.

| View Name | Purpose |
|-----------|---------|
| `contract_current` | Active contracts with owner, status, product association, and current version number |
| `contract_current_interface` | Active interfaces with validation status, SLA thresholds, and last validated timestamp |
| `contract_current_field` | Active fields with contracted types, nullability, classification, and breaking-change rule |
| `contract_current_rule` | Active rules with severity, threshold, and last result status |

**Example — `contract_current_interface`:**

```sql
REPLACE VIEW {Product}_Semantic_V.contract_current_interface AS
LOCKING ROW FOR ACCESS
SELECT
     c.contract_key
    ,c.contract_name
    ,c.owner_name
    ,cv.version_number
    ,ci.interface_name
    ,ci.interface_type
    ,ci.database_name
    ,ci.object_name
    ,ci.refresh_frequency
    ,ci.freshness_sla_minutes
    ,ci.availability_sla_pct
    ,ci.validation_status
    ,ci.last_validated_dts
FROM
    {Product}_Semantic.contract AS c
INNER JOIN {Product}_Semantic.contract_version AS cv
    ON cv.contract_id = c.contract_id
   AND cv.version_status = 'ACTIVE'
INNER JOIN {Product}_Semantic.contract_interface AS ci
    ON ci.contract_version_id = cv.contract_version_id
   AND ci.is_active = 1
WHERE
    c.is_active = 1
;
```

---

### 12.2 Evidence Views — `{Product}_Observability_V`

**Purpose:** Surface the evidence of whether contracts were met.

| View Name | Purpose |
|-----------|---------|
| `contract_latest_validation` | Most recent validation run result per interface — current health status |
| `contract_validation_history` | Rolling window of validation history (configurable period, e.g., 30 days) — trend data |
| `contract_violation_summary` | Open violations grouped by severity and consumer impact — incident management input |
| `contract_sla_status` | Current SLA status per interface — freshness and timeliness at a glance |

---

### 12.3 Consumer Impact Views — `{Product}_Observability_V`

**Purpose:** Answer the question "who is affected by an open violation?"

| View Name | Purpose |
|-----------|---------|
| `contract_registered_consumer` | All active consumers with version, notification channel, and classification access |
| `contract_consumer_impact` | Open violations joined to registered consumers — identifies who needs to be notified and at what severity |
| `contract_deprecated_consumer` | Consumers still registered against a `DEPRECATED` contract version — escalation input |

---

### 12.4 Governance Views — `{Product}_Memory_V`

**Purpose:** Support audit, change control, and sign-off tracking.

| View Name | Purpose |
|-----------|---------|
| `contract_change_history` | Full audit trail of all changes across all contracts and versions |
| `contract_signoff_status` | Pending and completed consumer sign-off records for MAJOR version changes |
| `contract_breaking_change_summary` | Planned MAJOR changes with affected consumer count and sign-off completion status |

---

### 12.5 Standards Export Views — `{Product}_Semantic_V`

**Purpose:** Shape contract data for consumption by external standards tooling. Each view produces output aligned to the corresponding standard's schema.

| View Name | Target Standard |
|-----------|----------------|
| `contract_odcs_export_v` | ODCS — produces a per-interface JSON structure mapping to the ODCS specification |
| `contract_dpds_port_export_v` | DPDS — shapes each interface as a DPDS output port descriptor |
| `contract_dcat_dataset_export_v` | DCAT — produces JSON-LD Dataset and Distribution records |
| `contract_lineage_context_v` | OpenLineage — provides the join context needed to add contract facets to lineage events |
| `contract_telemetry_context_v` | OpenTelemetry — provides the contract attribute values for telemetry span enrichment |

---

## 13. Interactive Contract Dashboards

The contract view groups are designed to feed human-consumable dashboards. Because all products built to this standard expose the same view groups with the same column contracts, a single dashboard template can serve any compliant product without modification.

### 13.1 Contract Health Dashboard

**Purpose:** Per-product, per-interface traffic-light status.

**Primary views:** `contract_current_interface`, `contract_latest_validation`, `contract_sla_status`.

**Key metrics displayed:**
- Interface validation status (PASSING / FAILING / WARNING / UNKNOWN) — traffic light per interface
- Freshness status (FRESH / STALE / CRITICAL) per interface
- Active violation count by severity
- Days since last successful validation per interface

**Audience:** Data product owners, data stewards, on-call engineers.

---

### 13.2 Consumer Impact Dashboard

**Purpose:** Which consumers are affected by open violations?

**Primary views:** `contract_consumer_impact`, `contract_registered_consumer`, `contract_violation_summary`.

**Key metrics displayed:**
- Count of open violations by consumer
- Highest violation severity affecting each consumer
- Notification channel per consumer (to confirm notifications were routable)
- Time since violation was opened (ageing indicator)

**Audience:** Contract owners, consumer team leads, incident managers.

---

### 13.3 SLA Compliance Dashboard

**Purpose:** Freshness and timeliness trending over time.

**Primary views:** `contract_sla_status`, `contract_validation_history`.

**Key metrics displayed:**
- Freshness lag trending (hours since last refresh, rolling 30-day view)
- SLA breach count over time
- Timeliness compliance rate (percentage of refresh cycles meeting the schedule)
- Interfaces approaching SLA threshold (within 80% of limit)

**Audience:** Data product owners, SRE / platform teams, SLA reporting consumers.

---

### 13.4 Change Control Dashboard

**Purpose:** Pending changes, sign-off status, deprecation timelines.

**Primary views:** `contract_breaking_change_summary`, `contract_signoff_status`, `contract_change_history`.

**Key metrics displayed:**
- Upcoming MAJOR changes with effective date
- Sign-off completion percentage per MAJOR change
- Consumers still pending sign-off (with days to deadline)
- Consumers still on deprecated versions (with days to retirement)

**Audience:** Data product owners, consumer team leads, governance teams.

---

### 13.5 Regulatory Evidence Dashboard

**Purpose:** Audit-ready evidence for a specified date range.

**Primary views:** `contract_validation_history`, `contract_change_history`, `contract_signoff_status`, `contract_sla_status`.

**Key metrics displayed:**
- Validation pass rate for each interface over the audit period
- SLA compliance rate over the audit period
- All breaking changes within the period with sign-off evidence
- Open violations within the period with remediation status and resolution date

**Audience:** Internal audit, compliance teams, external regulators.

---

## 14. Multi-Consumer View Pattern

### 14.1 One Product, Many Promises

Different consumers frequently need different contractual promises from the same underlying data. A branch banking system needs near-real-time balances. A regulatory reporting system needs certified end-of-day balances that cannot change after close. A fraud detection system needs raw signals, not business-filtered data. A marketing system needs consent-filtered, campaign-safe data.

Serving all of these from a single contracted interface would require the contract to be the weakest common denominator — the loosest SLA, the lowest classification, and the most permissive business logic. That produces a contract that is nearly meaningless.

**The correct pattern** is: one data product → multiple governed interfaces → each interface exposed as a view → each view has its own contract → consumers registered against the contract and version appropriate to their use case.

### 14.2 Decision Rule for Separate Interfaces

Create a separate contracted interface when consumers differ on any of the following:

| Dimension | Example Difference |
|-----------|-------------------|
| **Security / classification** | Branch operations sees full balance; marketing sees masked account number |
| **Business meaning** | Operational balance vs. reconciled regulatory balance — different business definition |
| **Freshness / SLA** | 15-minute refresh for operations vs. once-daily certified snapshot for regulatory |
| **Quality / certification** | Uncertified intraday vs. reconciled and immutable end-of-day |
| **Grain or aggregation** | Account-level for operations vs. portfolio-level for management reporting |
| **Lifecycle stability** | Stable certified interface vs. experimental analytical interface |

### 14.3 Example: Account Balance Product

The Account Balance data product exposes five contracted interfaces, each with its own contract, SLA, and consumer set:

| Interface | Consumers | Key Promises |
|-----------|-----------|-------------|
| `account_balance_current` | Branch banking, digital banking | 15-minute freshness; full balance; CONFIDENTIAL classification |
| `account_balance_regulatory_eod` | Finance regulatory reporting, RBA submissions | Daily end-of-day certified; immutable after close; RESTRICTED classification |
| `account_balance_fraud_near_real_time` | Fraud detection engine | Sub-5-minute freshness; raw transaction signals; CONFIDENTIAL |
| `account_balance_marketing_eligible` | Campaign management, CRM | Daily; consent-filtered; marketing-safe balance bands only; INTERNAL |
| `account_balance_deidentified` | Data science, model training | Daily; k-anonymised; no direct identifiers; INTERNAL |

Each interface has its own `contract`, `contract_version`, `contract_interface`, and `contract_rule` records in the Semantic module. Each has its own `contract_sla_status` and `contract_violation` tracking in Observability. Each has its own Access layer role following the naming convention in Section 9.2.

---

## 15. Designer Responsibilities

### 15.1 Responsibilities Checklist

The following checklist must be completed before any data product interface is deployed to production.

**Contract Definition (Semantic module):**

- [ ] Identified all public-facing output interfaces for this product (see Section 3.2 for the definition of "public-facing")
- [ ] Created a `contract` row for each contractable interface group
- [ ] Created a `contract_version` row at version `1.0.0` with `effective_from_dt` set
- [ ] Created a `contract_interface` row for each physical view or table covered by the contract
- [ ] Created `contract_field` rows for every field in each contracted interface, with `data_classification` and `breaking_change_rule` set
- [ ] Created at least one `contract_rule` per interface covering freshness and a minimum of one quality dimension (completeness, validity, or uniqueness)
- [ ] Registered all known consumers in `contract_consumer` against the `1.0.0` version

**Validation (Observability module):**

- [ ] Configured the validation pipeline to run on each refresh cycle and write results to `contract_validation_run` and `contract_validation_result`
- [ ] Verified that `contract_violation` rows are created for enforced rule failures
- [ ] Verified that `contract_sla_status` is updated after each pipeline run

**Alerting:**

- [ ] Configured violation routing to the organisation's operational channels (PagerDuty, ServiceNow, Slack, email) per the severity matrix in Section 8.3
- [ ] Verified that `notification_channel` is populated for all registered consumers

**Governance (Memory module):**

- [ ] Created at least one `contract_design_decision` record documenting key contract design choices (versioning strategy, SLA rationale, consumer scope)
- [ ] Created an initial `contract_change_log` entry for the `1.0.0` release
- [ ] If an exception was granted under Section 3.3, created a `contract_design_decision` record with `decision_category = 'CONTRACT_EXCEPTION'`

**Access layer:**

- [ ] Created Access layer views in `{Product}_Access_V` — no direct grants to Domain tables
- [ ] Named roles following the `{product}_{interface}_{accesslevel}_role` convention
- [ ] Verified that field classification in `contract_field` is reflected in view column selection or masking

**External standards:**

- [ ] Added the product to the relevant catalogue exports (ODCS files in `/contracts/`, DataHub publish config, DCAT export)
- [ ] Verified that `openlineage_job_name` and `openlineage_namespace` in `data_lineage` are populated for all contracted interface pipelines, enabling contract facet enrichment

---

## Appendix: Quick Reference

### Module Responsibilities

```
Semantic     → contract definition, versioning, field schema, rules, consumer registration
Observability → validation runs, rule results, violations, SLA status
Memory       → change log, design decisions, consumer sign-off records
Domain       → physical data exposed through contracted interfaces
Access       → roles and grants tied to contract interfaces and versions
```

### Lifecycle

```
DEFINE → VERSION → REGISTER → VALIDATE → ENFORCE → ALERT → EVIDENCE → CHANGE-CONTROL
```

### Violation Severity

```
CRITICAL  → 4 hour SLA    → PagerDuty + ServiceNow + consumer notification
HIGH      → 24 hour SLA   → ServiceNow + consumer notification
MEDIUM    → 5 business days → ServiceNow + owner notification
LOW       → 14 business days → Slack/email digest
INFO      → No remediation → Audit log only
```

### Versioning

```
MAJOR  → breaking change  → consumer sign-off required → minimum notice period applies
MINOR  → additive change  → no sign-off required
PATCH  → correction only  → no sign-off required
```

### Role Naming

```
{product}_{interface}_{accesslevel}_role
e.g. custmgmt_customer_current_read_role
```

---

## Document Change Log

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2026-05-11 | Initial Data Contract Design Standard. Establishes the cross-cutting contract backbone across Semantic, Observability, Memory, Domain, and Access modules. Defines the 8-step lifecycle, semantic versioning rules, consumer registration and sign-off, violation event types and severity levels, access role naming tied to contract versions, integration with ODCS, DPDS, ODPS, DCAT, DataHub, OpenMetadata, OpenLineage, and OpenTelemetry, repository structure, view groups, dashboard patterns, multi-consumer interface pattern, and designer responsibilities checklist. Implements all 8 gap analysis recommendations. | Worldwide Data Architecture Team, Teradata |

---

**End of Data Contract Design Standard**
