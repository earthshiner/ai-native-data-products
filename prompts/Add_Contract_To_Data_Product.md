# AI-Native Data Product — Add Contract Backbone Prompt
## For Adding the Data Contract Design Standard to an Existing Data Product

---

## How to Use This Prompt

1. Copy this entire prompt
2. Replace all `[PLACEHOLDER]` values with your specifics
3. Paste into a new Claude conversation
4. Answer the opening questions — then let Claude drive one deliverable at a time

> **Note**: This prompt is designed for use with the AI-Native module design skills installed.
> The Data Contract Design Standard is the authoritative reference for all contract design.
> Claude will draw on it automatically — no need to attach it separately.

---

## Context

The Data Contract Design Standard defines a **cross-cutting contract backbone** that spans
the existing six modules — it is not a seventh module. The backbone:

- Stores contract definitions in the **Semantic** module
- Records validation evidence in the **Observability** module
- Holds governance history and sign-off in the **Memory** module
- Exposes contracted data through **Domain** / **Access** views
- Controls consumption via **Access** layer roles and publication gates

This prompt guides a designer through adding that backbone to an **existing** data product
that was built before the contract standard was adopted.

---

## Starter Prompt

You are collaborating with a Data Architecture expert at Teradata.

Your role is to help add the **Data Contract Design Standard** to the **[DATA_PRODUCT_NAME]**
data product. This product is already deployed and follows the AI-Native six-module
architecture. We are **not** redesigning the product — we are adding the contract backbone
on top of the existing modules.

**Apply the Data Contract Design Standard exactly. Document any product-specific decisions
as `contract_design_decision` records with `decision_category = 'CONTRACT_EXCEPTION'`.**

---

### This Data Product

**Product name:** [DATA_PRODUCT_NAME]
(Databases follow the pattern: `[DATA_PRODUCT_NAME]_Semantic`, `[DATA_PRODUCT_NAME]_Observability`, etc.)

**Primary public-facing interfaces already deployed:**
[List the existing views or tables that external consumers query, e.g.:
- `[DATA_PRODUCT_NAME]_Access_V.customer_current`
- `[DATA_PRODUCT_NAME]_Access_V.account_balance_daily`
]

**Known consumers:**
[Name each consumer system, team, or application that uses the above interfaces, and its purpose]

**Environments:**
[e.g., DEV (host: dev-td), TEST (host: tst-td), PRD (host: prd-td)]

**Regulatory context:**
[Any compliance frameworks that apply, e.g. APRA CPG 235, SOX, GDPR — affects tag classification
and whether `require_change_ref` should be enforced in the PRD environment]

**Existing pipeline tooling:**
[What runs the ETL/refresh pipelines? ADF, Informatica, dbt, Python scripts, etc.
The validation pipeline will integrate with this.]

---

### Design Sequence

Work through the deliverables below **in order**.
**Stop after each deliverable, present the output, and wait for review before continuing.**

Group clarifying questions — no more than 3–4 at a time.

---

#### Deliverable 1 — Interface Audit and Contract Scope

- List every view and table in the `[DATA_PRODUCT_NAME]_Access_V` database
- For each, confirm: (a) is it public-facing under the standard definition? (b) does it need
  a contract?
- Identify any interface that qualifies for a contract exception (Section 3.3 of the standard)
  and record the reason
- Propose the `contract_key` for each contracted interface (e.g., `customer_current_v1`)
- Identify the `publication_gate_mode` for each interface — `BLOCKING` for any interface used
  in production ETL pipelines, AI model inputs, regulatory reporting, or channel-facing data;
  `ADVISORY` for exploratory or development interfaces
- Identify any consumers that are not yet registered and will need `contract_consumer` entries

*Stop here and wait for review.*

---

#### Deliverable 2 — Contract Definition DDL (Semantic Module additions)

For each contracted interface identified in D1, produce:

**`contract` rows** — one per logical contract group (not per interface):
- `contract_key`, `contract_name`, `contract_description`, `product_name`
- `owner_name`, `owner_contact`
- `license_uri` if data is shared outside the immediate product team
- `odcs_artefact_path` (e.g., `/contracts/customer_current.odcs.yaml`)

**`contract_version` rows** — one per contract at version `1.0.0`:
- `version_number = '1.0.0'`, `version_status = 'ACTIVE'`
- `compatibility_level = 'NON_BREAKING'` (initial release)
- `effective_from_dt = CURRENT_DATE`

**`contract_interface` rows** — one per contracted view/table:
- `database_name`, `object_name`, `interface_type` (usually `VIEW`)
- `interface_description` — one sentence explaining what the interface provides and to whom
- `refresh_frequency`, `refresh_frequency_uri` (Dublin Core vocabulary)
- `freshness_sla_minutes` — derive from existing pipeline schedule plus a buffer
- `publication_gate_mode` — from D1 analysis
- `publication_status = 'UNPUBLISHED'` — the first validation cycle will promote it

**`contract_field` rows** — one per column in each contracted interface:
- `field_name`, `data_type`, `is_nullable`, `data_classification`
- `is_pii`, `is_sensitive`
- `breaking_change_rule` — set `NEVER_REMOVE` for primary keys and business keys;
  `VERSION_BEFORE_CHANGE` for all other contractually meaningful fields

**`contract_consumer` rows** — one per known consumer per interface:
- `consumer_name`, `consumer_owner`, `consumer_purpose`
- `notification_channel` — must be monitored within 24 hours
- `data_classification_access`

**`contract_server` rows** — one per environment per interface:
- `server_id`, `environment`, `server_type = 'TERADATA'`
- `host`, `database_name_override` where applicable

**`contract_stakeholder` rows** — minimum OWNER + STEWARD for each contract

**`contract_tag` rows** — domain, subject area, and any applicable regulatory tags

*Stop here and wait for review.*

---

#### Deliverable 3 — Contract Floor Rules DDL (Semantic Module additions)

For each contracted interface, produce `contract_rule` rows for the **mandatory contract floor**
(Section 15.2 of the standard). Every production interface must have all four:

1. **`FRESHNESS_MAX_AGE`** — `rule_type = 'FRESHNESS'`, `rule_severity = 'CRITICAL'`,
   `gates_publication = 1`. Set `threshold_value` from the `freshness_sla_minutes` agreed in D2.

2. **`ROW_COUNT_MIN`** — `rule_type = 'ROW_COUNT'`, `rule_severity = 'CRITICAL'`,
   `gates_publication = 1`. Derive from the current row count with a conservative lower bound
   (e.g., 80% of the current minimum observed over the past 30 days).

3. **`NULL_RATE_MAX` on primary key field(s)** — `rule_type = 'QUALITY'`, `rule_severity = 'CRITICAL'`,
   `gates_publication = 1`, `threshold_value = 0.0`.

4. **`SCHEMA_VALIDATION`** — `rule_type = 'SCHEMA'`, `rule_severity = 'CRITICAL'`,
   `gates_publication = 1`. The `rule_logic` should describe the schema comparison check.

Then propose any **additional recommended rules** (Section 15.2.2) appropriate for each interface:
`NULL_RATE_MAX` for mandatory business fields, `UNIQUENESS` for business keys, `VALUE_SET` for
coded fields with a controlled vocabulary.

For each rule, document:
- `rule_name` (e.g., `customer_current_freshness_sla`)
- `rule_logic` (SQL expression or prose description of the check)
- `threshold_value`, `threshold_operator`, `threshold_unit`
- `is_enforced = 1` (all floor rules must be enforced)

*Stop here and wait for review.*

---

#### Deliverable 4 — Publication Gate: Access Layer View Updates

For each contracted interface where `publication_gate_mode = 'BLOCKING'`, produce an updated
`REPLACE VIEW` statement that adds the publication gate `EXISTS` check.

Use the Teradata pattern from Section 5.3.6 of the standard:

```sql
REPLACE VIEW [DATA_PRODUCT_NAME]_Access_V.{interface_name}
(
     {column_list}
)
AS
LOCKING ROW FOR ACCESS
SELECT
    {column_expressions}
FROM {domain_source} AS d
WHERE {existing_filters}
  AND EXISTS (
    SELECT 1
    FROM [DATA_PRODUCT_NAME]_Semantic.contract_interface AS ci
    WHERE ci.database_name      = '[DATA_PRODUCT_NAME]_Access_V'
      AND ci.object_name        = '{interface_name}'
      AND ci.publication_status = 'PUBLISHED'
      AND ci.is_active          = 1
)
;
```

**Include the explicit column list** before `AS` — every column the view exposes, one per line,
leading-comma style. Agents must be able to discover the view's columns from the DDL
without parsing the SELECT clause.

If the existing view does not have a column list, add one now.

*Stop here and wait for review.*

---

#### Deliverable 5 — Observability Module Additions

Confirm that the Observability module database (`[DATA_PRODUCT_NAME]_Observability`) exists
and has the standard observability tables. Then produce:

- `CREATE TABLE` statements for the four contract evidence tables:
  `contract_validation_run`, `contract_validation_result`, `contract_violation`, `contract_sla_status`
- `COMMENT ON TABLE` and `COMMENT ON COLUMN` for all new tables
- The seed `INSERT` statement to initialise `contract_sla_status` for each contracted interface
  with `freshness_status = 'UNKNOWN'` and `timeliness_status = 'UNKNOWN'`

Produce the **validation pipeline template** — a SQL script or procedure outline that:
1. For each contracted interface: select all `ACTIVE` rules from `contract_rule`
2. Execute each rule check and insert into `contract_validation_run` and `contract_validation_result`
3. Update `contract_interface.validation_status` and `contract_interface.last_validated_dts`
4. For `gates_publication = 1` rules: update `publication_status` per Section 5.3.5
5. Update `contract_sla_status.freshness_status`

*Stop here and wait for review.*

---

#### Deliverable 6 — Memory Module Additions

Confirm that the Memory module database (`[DATA_PRODUCT_NAME]_Memory`) exists.
Produce `INSERT` statements for:

**`contract_change_log`** — one entry per contracted interface recording the initial
contract adoption:
- `change_type = 'CONTRACT_ADOPTED'`
- `change_description` — brief narrative of why the contract backbone was added
- `breaking_change_ind = 0`
- `migration_notes` — note that consumers should register

**`contract_design_decision`** — minimum entries:
- `DD-CONTRACT-001` — choice of `publication_gate_mode` for each interface (BLOCKING/ADVISORY), with rationale
- `DD-CONTRACT-002` — freshness SLA derivation approach for each interface
- One entry per contract exception granted under Section 3.3 (if any)
- One entry per interface where `ADVISORY` was chosen instead of `BLOCKING`, recording the reason

*Stop here and wait for review.*

---

#### Deliverable 7 — ODCS Artefact Generation

For each contracted interface, produce the ODCS YAML artefact (`/contracts/{interface_name}.odcs.yaml`)
using the source tables populated in D2–D3. The ODCS file should map to the standard fields
defined in Section 10.1 of the Data Contract Design Standard:

- `info` section: contract identity, version, owner, contact
- `schema` section: all contracted fields with types, nullability, classification
- `quality` section: all contract rules with thresholds and severity
- `sla` section: freshness_sla_minutes, availability_sla_pct
- `servers` section: connection details per environment
- `consumers` section: registered consumers and purpose
- `tags` section: all contract_tag entries

*Stop here and wait for review.*

---

#### Deliverable 8 — External Standards Export Views

For each interface, confirm that the five standards export views defined in Section 12.5
of the Data Contract Design Standard are created:

- `contract_odcs_export` — per-interface JSON for ODCS tooling
- `contract_dpds_port_export` — per-interface DPDS output port descriptor
- `contract_dcat_dataset_export` — JSON-LD Dataset and Distribution records
- `contract_lineage_context` — join context for OpenLineage facets (includes `datahub_dataset_urn`)
- `contract_telemetry_context` — attribute values for OpenTelemetry span enrichment

For each view: confirm the column list matches the standard, and confirm the `datahub_dataset_urn`
derivation pattern is correct for this product's Teradata environment.

*Stop here and wait for review.*

---

#### Deliverable 9 — Deployment Plan and Validation

Produce:

- **Deployment sequence** — table creation order respecting FK dependencies, then view updates
- **Pre-deployment checklist** (from Section 15.1 of the standard) — filled in for this product
- **First-run validation script** — execute the validation pipeline from D5, then query
  `contract_interface.publication_status` to confirm at least one interface reaches `PUBLISHED`
- **Rollback note** — if the publication gate causes consumer failures, the gate can be set to
  `ADVISORY` temporarily by setting `publication_gate_mode = 'ADVISORY'` on the affected interface;
  document this escape hatch as a `contract_design_decision` entry

*Final deliverable — present for sign-off.*

---

### Design Principles

- **Contracts are retroactive governance** — the product is not being redesigned; the contract is
  being added to what already exists. Start by describing reality accurately, then tighten.
- **Floor first, build up** — deploy the four mandatory contract floor rules before adding
  additional quality rules. Get the baseline running, then iterate.
- **BLOCKING by default** — production interfaces should use `BLOCKING` mode. Use `ADVISORY`
  only for genuinely exploratory interfaces, and document the choice.
- **Consumers are stakeholders** — do not deploy contracts without notifying consuming teams.
  They need to understand the publication gate and what it means when the interface returns no rows.
- **The contract describes reality** — set `freshness_sla_minutes` to what the pipeline actually
  delivers, not what you wish it delivered. Tighten it once you have baseline validation data.

---

### Let's Begin

Start with **Deliverable 1 — Interface Audit and Contract Scope**.

Before writing anything, ask me the clarifying questions you need to identify all public-facing
interfaces and their appropriate gate mode. Keep questions grouped — no more than 3–4 at a time.
