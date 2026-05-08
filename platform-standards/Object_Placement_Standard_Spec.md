# Object Placement Standard — Interface Specification

**Version:** 1.0  
**Status:** Normative  
**Scope:** Any relational or cloud data platform  
**Maintained by:** Paul Dancer — Ecosystem Architect - Teradata Worldwide Field Tech

---

## Purpose

This document defines the interface that any **Object Placement Standard** must satisfy for use with the AI-Native Data Product framework.

An AI-Native Data Product agent **must not** make assumptions about how an organisation structures its data containers. It **must** read a conforming implementation of this spec before generating any DDL, object placement decision, or access control statement.

This spec is **platform-agnostic**. It uses abstract terms (container, namespace, access principal) that map to platform-specific constructs in each implementation. It does not prescribe naming conventions, SQL dialects, or organisational structure — it only prescribes what must be declared.

---

## Terminology

| Term | Meaning |
|---|---|
| **Container** | The smallest named unit of object ownership on the platform. Maps to: Teradata DATABASE, Snowflake SCHEMA, BigQuery DATASET, Databricks SCHEMA, etc. |
| **Namespace** | A hierarchical grouping of containers. May be implicit (Teradata) or explicit (Databricks catalog.schema). |
| **Object** | A first-class data artefact: table, view, procedure, macro, function, index. |
| **Object type** | The kind of object. Implementations must declare which types they recognise and how they abbreviate them. |
| **Parent container** | A container that allocates space or organises child containers but holds no objects itself. Not all platforms have this concept. |
| **Child container** | A container that holds objects. On platforms with no parent concept, all containers are child containers. |
| **Access principal** | An identity that can be granted access rights. Maps to: Teradata ROLE or USER, Snowflake ROLE, BigQuery IAM principal, etc. |
| **Separation policy** | The rule governing whether different object types are co-located in the same container or separated into distinct containers. |
| **Structural container** | A container that exists solely to organise administrative scope, space allocation, or namespace grouping. It holds no data objects and no child containers that directly hold objects. Structural containers sit above the standard parent/child hierarchy. Not all platforms have this concept — implementations must declare whether they use it. |
| **Derivation function** | The deterministic algorithm that computes the correct target container name for a given object, given a set of input parameters. |
| **Validation procedure** | The platform-specific mechanism that confirms objects have been placed in the correct containers after generation. |
| **Implied grant** | An access permission that is not granted to an end-user principal directly, but is required for the separation architecture to function correctly — for example, a cross-container permission that allows a view in one container to reference objects in another. |

---

## Agent Consumption Instructions

When generating any object for a data product:

1. **Locate** a conforming implementation of this spec using the priority order below.
2. **Read all required sections** before generating any DDL statement, access grant, or access revocation.
3. **For each object**, call the Derivation Function with the object's type and context to determine its target container.
4. **Generate** the object in that container only. Never place objects in a structural container, a parent container, or in a container designated for a different object type.
5. **Provision implied grants** — if the implementation declares implied grants required by the separation architecture, provision them as part of the standard sequence, not as an afterthought.
6. **Run** the Validation Procedure after generation. If it fails, halt and report — do not proceed with dependent objects.
7. **Never assume** co-location of object types. Never default to a flat structure. If uncertain, ask.

**Priority order for locating an implementation:**

1. An explicit file path provided in the current conversation or project instructions
2. `/mnt/skills/user/object-placement/SKILL.md`
3. A file matching `Object_Placement_Standard*.md` in the project context
4. **If none of the above exist:** STOP. Do not proceed. Ask the user:

   > *"Before I generate any objects, I need your Object Placement Standard. Where should tables, views, and other object types be placed? Are they separated into distinct containers, or co-located? What is the naming convention for those containers?"*

---

## Required Sections

Every conforming implementation **MUST** include all eight of the following sections, using these exact headings. An agent may reject an implementation that omits any required section.

---

### Section 1 — Platform Declaration

**What it must contain:**

- The target platform name and version (if version-specific)
- The platform's native term for "container" (e.g. DATABASE, SCHEMA, DATASET)
- The platform's native term for "access principal" (e.g. ROLE, IAM Policy)
- Whether the platform uses a hierarchical namespace (e.g. catalog → schema → table) or a flat namespace
- The maximum container name length enforced by the platform
- Any reserved characters or words that must be avoided in container names

**Why:** Agents must know the platform's constraints before generating any names or DDL.

---

### Section 2 — Container Model

**What it must contain:**

- Whether the implementation uses structural containers, parent containers, child containers, or any combination
- If structural containers exist: what they hold (space allocation, administrative scope, namespace grouping) and what they must never hold (data objects)
- If parent containers exist: what they hold (space allocation, permissions) and what they must not hold (data objects)
- If neither structural nor parent containers exist: state that explicitly — all containers are child containers
- The full depth of the container hierarchy, counting all levels including structural containers (e.g. four levels: structural root → structural group → parent → child; or two levels: parent → child; or one level: flat)
- The rule for which containers may hold data objects and which may not
- Whether physical system boundaries exist between lifecycle phases (e.g. development and production on separate systems) — and if so, the implication for agent connectivity and DDL execution scope

**Why:** Agents must not place objects in structural containers, parent containers, or containers not designated for object ownership. On systems with physical boundaries, agents must not assume cross-system connectivity.

---

### Section 3 — Naming Pattern

**What it must contain:**

- The complete naming pattern expressed as an ordered sequence of segments, using `{{SegmentName}}` notation
- For each segment:
  - Its position in the pattern (1-based)
  - Its data type (single character, fixed-length numeric, fixed-length alphabetic, variable-length with max)
  - Its permitted values or the rule for deriving them
  - Whether it is mandatory or conditional (e.g. only present in certain lifecycle phases)
  - The separator character used between it and the next segment (underscore is conventional; implementations must state their choice)
- A worked example of a complete container name with each segment identified
- The ordering principle: why segments appear in this order (the spec recommends ordering from most-inclusive to least-inclusive to preserve semantic hierarchy, but does not mandate it)

- The environment-agnostic naming principle: object names must be **stable across all lifecycle phases**. Only the container changes between environments — object names must never carry environment markers (e.g. `_dev`, `_uat`, `_test` suffixes on object names are prohibited). Code promotion is a container substitution, not a rename.

**Why:** Agents parse container names deterministically. A fully-specified pattern enables both generation and validation without external lookups. Environment-agnostic object names ensure that promotion between phases requires no object renaming — only the target container changes.

---

### Section 4 — Object Placement Rules

**What it must contain:**

- A table mapping each recognised object type to its designated container suffix or container name rule
- An explicit statement of the separation policy: co-located, strictly separated, or conditionally separated
- For each object type: the abbreviated identifier used in container names (if applicable)
- Any object types that are explicitly excluded (e.g. temporary objects that do not persist in a named container)

**Minimum required object types an implementation must address:**

| Object type | Must be addressed |
|---|---|
| Persistent tables | MUST |
| Views | MUST |
| Stored procedures | MUST |
| Functions / UDFs | MUST |
| Macros / scripts | SHOULD |
| Indexes (if named separately) | SHOULD |
| Temporary / transient objects | SHOULD |

**Object naming rules — required declaration**

The implementation must declare which of the following naming rules applies, determined by its separation policy. The two rules are mutually exclusive:

**Rule A — Container-discriminated naming (applies when policy is `STRICT_SEPARATION`)**

When object types are placed in separate containers, the container name is the sole type discriminator. Object names must therefore be **identical across container types** for the same logical object. Type markers — prefixes such as `v_`, `t_`, or suffixes such as `_vw`, `_tbl`, `_view` — are **prohibited** on object names. They are redundant (the container already carries the type), they break name symmetry across containers, and they prevent deterministic cross-container navigation by name substitution alone.

An agent encountering a type-prefixed or type-suffixed object name in a `STRICT_SEPARATION` environment must flag it as a naming violation.

**Rule B — Name-discriminated naming (applies when policy is `CO_LOCATED` or `TYPE_GROUPED`)**

When object types share a container, the container cannot disambiguate type. The implementation must therefore declare an explicit disambiguation rule that produces a unique name for every `(logical_name x object_type)` combination within that container. The rule must be:
- Declared in full in the implementation — an agent must never infer or assume it
- Consistent — the same rule applied to every object of every type
- Reversible — given a disambiguated name, an agent can recover the logical name and the type

The implementation must document the disambiguation rule in this section and demonstrate it with at least two worked examples.

An agent must never apply a disambiguation convention it has not read from the implementation. If the implementation declares `CO_LOCATED` but provides no disambiguation rule, the implementation is non-conforming and the agent must halt and report.

**View-layer architecture — required declaration for implementations that include views**

If the implementation uses `STRICT_SEPARATION` and views exist alongside tables, it must declare whether a view-layer architecture applies — that is, whether views are divided into tiers with different rules governing which container types each tier may reference:

- If a view-tier architecture exists: the implementation must document the rules for each tier, which container types each tier may reference, and any DDL constraints that apply per tier (e.g. access locking, column list requirements). The number of tiers is unconstrained — implementations may define as many as the transformation architecture requires.
- If no view-tier architecture exists: state that explicitly.

An agent generating a view must know which tier it is working in before writing any DDL. The tier determines which containers the view may reference and which DDL patterns apply.

**Why:** This is the most critical section for correct agent behaviour. Without it, agents will co-locate objects that must be separated, breaking access control models. Without the naming rule declaration, agents working in co-located environments will invent disambiguation conventions inconsistently, producing unmaintainable and unparseable object names. Without a view-tier declaration, agents may incorrectly apply access-locking patterns to transformation views, or reference table containers from views that should only reference view containers.

---

### Section 5 — Separation Policy

**What it must contain:**

- An explicit declaration of one of the following policies:

  | Policy | Meaning |
  |---|---|
  | `STRICT_SEPARATION` | Each object type has its own designated container. Objects of different types are never in the same container. |
  | `TYPE_GROUPED` | Object types are grouped but not strictly separated (e.g. tables and indexes together, views separately). |
  | `CO_LOCATED` | All object types are placed in the same container. No separation by type. |
  | `CUSTOM` | A custom rule applies. The implementation must describe it fully. |

- The rationale for the chosen policy (one or two sentences)
- Any exceptions to the policy (e.g. temporary tables may share a container with work tables)
- The access implication of the policy: what access principals can reach, and what they cannot, as a direct consequence of this separation

**Why:** Separation policy is the primary mechanism by which access control is enforced at the container level. Agents must understand it to generate correct access principal assignments.

---

### Section 6 — Derivation Function

**What it must contain:**

- A deterministic, unambiguous algorithm for computing the correct target container name for any given object
- Expressed in pseudocode, structured natural language, or both — but it must be executable by an agent without further clarification
- The complete set of input parameters the function accepts
- Any conditional branches (e.g. integrated vs. workstream-level environments, production vs. development)
- At least three worked examples covering different branches of the algorithm
- The parent container derivation (if the platform uses parent containers) as a separate function or as a clearly identified step within the main function

**The function signature must accept at minimum:**

```
derive_container(
  object_type,        // the type of object to be placed
  environment_inputs, // platform-specific environment identifiers
  classification      // security or access classification, if applicable
) → container_name
```

**Why:** This is the function agents call before every CREATE statement. It must produce a unique, unambiguous answer for every valid input combination.

---

### Section 7 — Access Model

**What it must contain:**

- The access principal model: how access is granted to containers (role-based, user-based, policy-based, or a combination)
- Whether access is granted at the container level, the object level, or both — and which is preferred
- The standard access principal types and what each can do (e.g. read-only role, batch role, admin role)
- The rule for how access principals are named (if they follow a convention)
- Any explicit prohibitions (e.g. "no direct object-level grants to end users")
- How the separation policy interacts with access control (e.g. users with access to a view container cannot see the underlying table container)
- Any **implied grants** created by the separation architecture — access permissions that are not granted to end-user principals directly but are required for the architecture to function. For example: if strict separation places views in one container and their source objects in another, the view-installing user or database may require elevated cross-container access rights before any view can be created. The implementation must declare these implied grants explicitly, specify when they must be provisioned (before or after object creation), and include them in the standard provisioning sequence. An agent must never treat implied grants as optional.

**Why:** Agents generating data products must also generate correct access grants. Without an access model, they will either omit grants (producing inaccessible objects) or grant too broadly (producing a security violation). Implied grants created by strict separation are among the most commonly missed provisioning steps — they compile silently but fail at query time, producing error messages that appear to be view access problems rather than missing infrastructure.

---

### Section 8 — Validation Procedure

**What it must contain:**

- A platform-specific procedure, query, or checklist that confirms correct object placement after generation
- It must be executable by an agent without human intervention
- It must check at minimum:
  1. Objects exist in the containers they were intended for
  2. Objects are not present in containers designated for a different type
  3. Objects are not present in parent containers or structural containers
  4. End-user access principals have been granted at the correct level and not at a level that exposes containers they should not reach
  5. All implied grants required by the separation architecture are present — their absence is a provisioning defect even if all objects were created successfully
- The expected output of a passing validation (what "clean" looks like)
- The expected output of a failing validation (what an agent should report)
- The action an agent must take on failure: halt, report, and await instruction — never auto-correct silently

**Why:** Generation and placement are not sufficient. Agents must verify. A silent placement error produces a working but incorrectly structured system that is expensive to remediate.

---

## Optional Sections

Implementations MAY include any of the following. Agents should read them if present.

| Section | Purpose |
|---|---|
| **Environment lifecycle** | How containers change across development, test, and production phases. Useful for agents generating environment-specific DDL. |
| **Container retirement procedure** | How containers are retired when a project or workstream ends — including access revocation, object removal, and registry updates. |
| **Metadata and tagging** | Any required metadata attached to containers at creation time. |
| **Co-existence rules** | How this hierarchy standard co-exists with a legacy structure during migration. |
| **Automation hooks** | Scripts, APIs, or tools that implement or validate the standard programmatically. |

---

## Conformance Checklist

An implementation is conforming if it satisfies all of the following:

- [ ] Section 1 (Platform Declaration) is present and complete
- [ ] Section 2 (Container Model) is present and describes parent/child or flat structure
- [ ] Section 3 (Naming Pattern) is present and all segments are fully specified
- [ ] Section 4 (Object Placement Rules) covers all four mandatory object types (persistent tables, views, stored procedures, functions/UDFs) and declares a view-tier architecture or explicitly states that none applies
- [ ] Section 5 (Separation Policy) declares one of the four named policies
- [ ] Section 6 (Derivation Function) produces an unambiguous container name for any valid input
- [ ] Section 7 (Access Model) specifies the access principal type and grant level
- [ ] Section 8 (Validation Procedure) is executable by an agent without human input
- [ ] All section headings match the required headings exactly (agents use these for lookup)
- [ ] No section refers to a specific SQL dialect or platform construct in a way that would cause it to fail on a different platform (the implementation document may be platform-specific; the spec sections in a meta-document may not be)

---

## Non-Goals

This spec does not prescribe:

- Any specific naming convention (that is the implementation's responsibility)
- Any specific SQL dialect or DDL syntax
- Any specific access control technology
- The number of containers in a hierarchy
- The number of environments or lifecycle phases
- How containers are provisioned (manual, automated, agent-driven)
- Physical system topology — whether development and production share a system or are physically separated. This is an infrastructure decision. The naming convention must work consistently across both topologies; how objects are transported between systems is outside the scope of this standard.

Those decisions belong to the organisation and are expressed in their implementation.

---

## Versioning

Implementations should declare which version of this spec they conform to, using the `Spec version` field in their header. Breaking changes to required sections will increment the major version. Additive changes (new optional sections) will increment the minor version.

