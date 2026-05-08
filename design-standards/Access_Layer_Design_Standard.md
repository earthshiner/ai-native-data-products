# Access Layer Design Standard
## AI-Native Data Product Framework

| **Version** | 1.0 |
|-------------|-----|
| **Date** | 2026-05-07 |
| **Status** | Active |
| **Related documents** | AI_Native_Data_Product_Master_Design |

---

## 1. Purpose

The Access Layer is the mandatory access control artefact of an AI-Native Data Product. It creates standard database roles and grants them SELECT access to each module's databases, making the data product discoverable and queryable by agents and consumers as each module is deployed.

Without an Access Layer, a correctly deployed data product is operationally invisible: every consumer — agents, dashboards, reporting tools, and analysts — will be denied access regardless of how completely the module databases are deployed.

---

## 2. Core Principle

**Consumers access the module database(s) that expose the public interface for that module.** In the standard `{ProductName}_{ModuleName}` deployment pattern, this is the module database itself (which contains both base tables and views). In deployments that separate base tables and views into distinct databases (e.g., `{ProductName}_{Module}_T` for tables and `{ProductName}_{Module}_V` for views), consumers should be granted access to the view-layer database only — not the base table database.

This standard uses the term **module access database** to refer to whichever database(s) consumers should be granted access to for a given module, acknowledging that this varies by deployment approach.

---

## 3. Standard Roles

Three roles are created per data product. Naming convention: `{ProductName}_ROLE_{TIER}`.

| Role | Intended consumers | Scope |
|------|--------------------|-------|
| `{ProductName}_ROLE_READ` | Analysts, BI tools, ad-hoc SQL users | SELECT on module access databases |
| `{ProductName}_ROLE_AGENT` | AI agents, MCP servers, automated tools | SELECT on module access databases |
| `{ProductName}_ROLE_ADMIN` | Data product owner, data steward | SELECT on all databases including any separate base table databases |

### 3.1 Why ROLE_AGENT is separate from ROLE_READ

`ROLE_AGENT` and `ROLE_READ` grant the same SELECT scope by default but are kept as distinct roles for three reasons:

1. **Independent lifecycle** — agent access can be suspended, extended, or revoked without affecting human analyst access.
2. **Selective extension** — some products may grant agents write-back capability (for example, INSERT to Memory module tables to record agent interactions). This extension can be applied to `ROLE_AGENT` without broadening `ROLE_READ`.
3. **Audit clarity** — agent-originated queries are separately auditable when the connecting user holds a distinct role.

---

## 4. Deployment Timing

The Access Layer deploys in two phases, interleaved with the module deployment sequence:

| Phase | Timing | Action |
|-------|--------|--------|
| **Phase 1.5** | Immediately after Phase 1 (Memory + Semantic) | Create roles; grant on Semantic and Memory module access databases |
| **Phase 2.5** | Immediately after Phase 2 (Domain + Observability) and as further modules deploy | Extend grants to Domain, Observability, and any subsequent module databases |

**Phase 1.5 is the minimum viable access grant.** Once Semantic and Memory are readable, agents can discover the data product's structure, read the business glossary, and use the query cookbook. Operational tools such as data product dashboards can begin populating.

Delaying all grants until every module is fully deployed is an anti-pattern: consumers cannot validate the product during incremental deployment, and access problems are discovered later than necessary.

---

## 5. Grant Matrix

| Module | ROLE_READ | ROLE_AGENT | ROLE_ADMIN |
|--------|-----------|------------|------------|
| Semantic | ✅ Phase 1.5 | ✅ Phase 1.5 | ✅ Phase 1.5 |
| Memory | ✅ Phase 1.5 | ✅ Phase 1.5 | ✅ Phase 1.5 |
| Domain | ✅ Phase 2.5 | ✅ Phase 2.5 | ✅ Phase 2.5 |
| Observability | ✅ Phase 2.5 | ✅ Phase 2.5 | ✅ Phase 2.5 |
| Search | ✅ when deployed | ✅ when deployed | ✅ when deployed |
| Prediction | ✅ when deployed | ✅ when deployed | ✅ when deployed |
| Base table databases (if separate) | ❌ | ❌ | ✅ |

Grants to Search and Prediction are applied as those modules are deployed, following the same Phase 2.5 pattern.

---

## 6. DCL Template (Teradata)

The following template targets the standard `{ProductName}_{ModuleName}` deployment pattern. Teams using a separate view-layer database (e.g., `{ProductName}_{Module}_V`) should substitute the appropriate database names. See the Teradata Platform Profile (`Advocated_Data_Management_Standards.md`) for platform-specific syntax notes.

```sql
-- =============================================================================
-- ACCESS LAYER — {ProductName} Data Product
-- File: 00-access/{ProductName}_access_layer.dcl
--
-- Phase 1.5: Apply after Memory + Semantic are deployed.
-- Phase 2.5: Apply after Domain + Observability are deployed.
--            Add further GRANT blocks as additional modules are deployed.
-- =============================================================================

-- ── Create roles ──────────────────────────────────────────────────────────────

CREATE ROLE {ProductName}_ROLE_READ;
COMMENT ON ROLE {ProductName}_ROLE_READ IS
    '{ProductName} data product — read-only access for analysts and BI tools.
     Grants SELECT on all module access databases.';

CREATE ROLE {ProductName}_ROLE_AGENT;
COMMENT ON ROLE {ProductName}_ROLE_AGENT IS
    '{ProductName} data product — AI agent and automated tool access.
     Grants SELECT on all module access databases. Kept separate from
     ROLE_READ to allow independent lifecycle management and selective
     extension (e.g., agent write-back to Memory).';

CREATE ROLE {ProductName}_ROLE_ADMIN;
COMMENT ON ROLE {ProductName}_ROLE_ADMIN IS
    '{ProductName} data product — data product owner and data steward access.
     Grants SELECT on all databases.';

-- ── Phase 1.5 — apply after Memory + Semantic are deployed ───────────────────

GRANT SELECT ON {ProductName}_Semantic TO {ProductName}_ROLE_READ;
GRANT SELECT ON {ProductName}_Semantic TO {ProductName}_ROLE_AGENT;
GRANT SELECT ON {ProductName}_Semantic TO {ProductName}_ROLE_ADMIN;

GRANT SELECT ON {ProductName}_Memory   TO {ProductName}_ROLE_READ;
GRANT SELECT ON {ProductName}_Memory   TO {ProductName}_ROLE_AGENT;
GRANT SELECT ON {ProductName}_Memory   TO {ProductName}_ROLE_ADMIN;

-- ── Phase 2.5 — apply after Domain + Observability are deployed ───────────────

GRANT SELECT ON {ProductName}_Domain        TO {ProductName}_ROLE_READ;
GRANT SELECT ON {ProductName}_Domain        TO {ProductName}_ROLE_AGENT;
GRANT SELECT ON {ProductName}_Domain        TO {ProductName}_ROLE_ADMIN;

GRANT SELECT ON {ProductName}_Observability TO {ProductName}_ROLE_READ;
GRANT SELECT ON {ProductName}_Observability TO {ProductName}_ROLE_AGENT;
GRANT SELECT ON {ProductName}_Observability TO {ProductName}_ROLE_ADMIN;

-- ── When Search is deployed ───────────────────────────────────────────────────

-- GRANT SELECT ON {ProductName}_Search      TO {ProductName}_ROLE_READ;
-- GRANT SELECT ON {ProductName}_Search      TO {ProductName}_ROLE_AGENT;
-- GRANT SELECT ON {ProductName}_Search      TO {ProductName}_ROLE_ADMIN;

-- ── When Prediction is deployed ───────────────────────────────────────────────

-- GRANT SELECT ON {ProductName}_Prediction  TO {ProductName}_ROLE_READ;
-- GRANT SELECT ON {ProductName}_Prediction  TO {ProductName}_ROLE_AGENT;
-- GRANT SELECT ON {ProductName}_Prediction  TO {ProductName}_ROLE_ADMIN;

-- ── Assign roles to users and service accounts ────────────────────────────────
-- Roles are product artefacts — created once, owned by the data product team.
-- User-to-role assignments are operational events.
-- Replace placeholders with the actual usernames for this environment.

-- AI agent / MCP server service account:
-- GRANT {ProductName}_ROLE_AGENT TO {agent_service_account};

-- Analyst or BI tool user (or grant to an existing organisational group role):
-- GRANT {ProductName}_ROLE_READ TO {analyst_user_or_group_role};

-- Data product owner / data steward:
-- GRANT {ProductName}_ROLE_ADMIN TO {product_owner_user};
```

---

## 7. Artefact Location

The DCL file lives in a `00-access/` directory at the root of the data product's artefact tree. The `00-` prefix marks it as a pre-requisite that must be visible alongside the module directories:

```
{ProductName}/
├── 00-access/
│   └── {ProductName}_access_layer.dcl
├── 01-memory/
├── 02-semantic/
├── 03-domain/
├── 04-observability/
├── 05-search/           (if implemented)
└── 06-prediction/       (if implemented)
```

The two-phase structure is documented in the file's comments. How the two phases are executed — whether as a single pass, two separate executions, or any other delivery mechanism — is left to the team deploying the product.

---

## 8. Required Documentation Record

Deployment of the Access Layer must produce a `Design_Decision` record in `{ProductName}_Memory.Design_Decision`. This is mandatory under the Documentation Capture Protocol in the Master Design Standard.

```sql
INSERT INTO {ProductName}_Memory.Design_Decision
(
    decision_id,
    decision_version,
    decision_title,
    decision_description,
    context,
    alternatives_considered,
    rationale,
    decision_status,
    decision_category,
    source_module,
    module_version,
    decided_date,
    valid_from,
    valid_to,
    is_current
)
VALUES
(
    'DD-ACCESS-001',
    1,
    'Three-tier role model for data product access control',

    'Three database roles are created per product:
     {ProductName}_ROLE_READ for analysts and BI tools,
     {ProductName}_ROLE_AGENT for AI agents and automated tools,
     {ProductName}_ROLE_ADMIN for the product owner and data steward.
     All consumer roles receive SELECT on the module access databases.
     ROLE_ADMIN additionally receives access to any separate base table
     databases.',

    'Consumers — agents, dashboards, and tools — require SELECT on the
     Semantic and Memory databases at minimum to discover and operate the
     data product. Without this, the product is physically deployed but
     operationally invisible to all consumers.',

    'Option 1 (chosen): Three distinct roles with separate READ and AGENT tiers.
     Option 2: Single consumer role — rejected because READ and AGENT access
     cannot then be independently managed or extended.
     Option 3: Direct per-user grants — rejected because this does not scale
     and prevents role-based revocation.',

    'Keeping ROLE_AGENT separate from ROLE_READ enables independent lifecycle
     management of AI tooling access and permits selective extension
     (e.g., agent write-back to Memory) without broadening analyst access.',

    'ACCEPTED',
    'SECURITY',
    'ACCESS',
    '1.0',
    CURRENT_DATE,
    CURRENT_DATE,
    DATE '9999-12-31',
    1
);
```

---

## 9. Relationship to Module Design Standards

Each module design standard defines **what** that module contains and **what** it registers in Semantic and Memory on deployment. The Access Layer defines **who** can read each module after deployment.

The roles themselves (`{ProductName}_ROLE_READ`, `_ROLE_AGENT`, `_ROLE_ADMIN`) are **data product artefacts** — created once when the product is first deployed and owned by the data product team.

The assignment of specific users or service accounts to those roles is an **operational event** — outside the scope of these design standards and managed through whatever access request and approval process the data product team operates.

---

## 10. Checklist

- [ ] `{ProductName}_ROLE_READ` created with COMMENT
- [ ] `{ProductName}_ROLE_AGENT` created with COMMENT
- [ ] `{ProductName}_ROLE_ADMIN` created with COMMENT
- [ ] Phase 1.5 grants applied (Semantic, Memory) immediately after Phase 1 deployed
- [ ] Phase 2.5 grants applied (Domain, Observability) immediately after Phase 2 deployed
- [ ] Search and Prediction grants applied as each is deployed
- [ ] Service accounts and users assigned to appropriate roles for this environment
- [ ] `DD-ACCESS-001` Design_Decision record inserted into `{ProductName}_Memory`
- [ ] DCL file committed as `00-access/{ProductName}_access_layer.dcl`

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-05-07 | Paul Dancer, Worldwide Data Architecture Team, Teradata | Initial Access Layer Design Standard. Established three-tier role model (ROLE_READ, ROLE_AGENT, ROLE_ADMIN), two-phase deployment timing (1.5 and 2.5), grant matrix, DCL template, artefact location convention, and required documentation record. Motivated by D01_MP deployment finding where all module databases were correctly deployed but all consumers received access denied (Error 3523), rendering the product operationally invisible. |
