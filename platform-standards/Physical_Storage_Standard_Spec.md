# Physical Storage Standard — Interface Specification

**Version:** 1.0  
**Status:** Normative  
**Scope:** Any platform using object storage as the physical layer beneath a logical data container model  
**Maintained by:** Paul Dancer — EcoSystem Architect, Teradata Worldwide Field Tech  
**Companion standard:** Object Placement Standard — Interface Specification v1.0

---

## Purpose

This document defines the interface that any **Physical Storage Standard** must satisfy when an organisation uses object storage (e.g. Amazon S3, Azure Data Lake Storage, Google Cloud Storage) as the physical layer beneath its logical data containers.

The Object Placement Standard governs the **logical layer** — container naming, object naming, access principals. This standard governs the **physical layer** — object store paths, file formats, partition strategies, physical access controls, and data lifecycle. The two standards are explicitly coupled: the physical path is derived deterministically from the logical names defined in the Object Placement Standard. Neither standard is complete without the other when object storage is in use.

This spec is **platform-agnostic**. It uses abstract terms (object store, path, bucket, partition) that map to platform-specific constructs in each implementation. It does not prescribe specific storage services, path structures, or file formats — it only prescribes what must be declared.

**Relationship to the Object Placement Standard:**

- The Object Placement Standard is the **primary standard**. It must exist and be conforming before a Physical Storage Standard can be written.
- This standard **extends** the Object Placement Standard for organisations using object storage. It does not replace any section of it.
- If an organisation uses only proprietary block storage (no object store), this standard does not apply. The Object Placement Standard alone is sufficient.
- If object storage is used for any subset of objects, this standard applies to that subset and must declare which objects it governs.

---

## Terminology

| Term | Meaning |
|---|---|
| **Object store** | A scalable, flat storage service accessed via key-value semantics. Keys are paths; values are binary objects (files). Maps to: Amazon S3, Azure Data Lake Storage (ADLS) Gen2, Google Cloud Storage (GCS), MinIO, etc. |
| **Bucket** | The top-level named container in an object store. All paths exist within a bucket. Bucket-level policies govern coarse-grained access. |
| **Path** | The full key used to address an object or a logical prefix (directory) in the object store. Equivalent to a file system path but with no native directory concept — the hierarchy is encoded in the key string. |
| **Path prefix** | The portion of a path that identifies a logical grouping — equivalent to a directory. All objects whose key begins with a given prefix belong to that logical group. |
| **Open Table Format (OTF)** | A table format specification layered over object store files that provides ACID transactions, schema evolution, time travel, and hidden partitioning. Examples: Apache Iceberg, Delta Lake, Apache Hudi. |
| **File format** | The binary format used to encode individual data files within an OTF table. Examples: Apache Parquet, Apache ORC, Apache Avro. |
| **Partition** | A logical subdivision of a table's data, used to improve query performance by pruning irrelevant files. May be Hive-style (encoded in the path) or hidden (managed by the OTF metadata, not in the path). |
| **Hive-style partitioning** | Partitioning where partition values are encoded as path segments in the form `key=value`. Readable by any tool that understands Hive metastore conventions. |
| **Hidden partitioning** | Partitioning managed entirely within the OTF metadata layer (e.g. Iceberg partition specs). The path does not encode partition values. Provides more flexibility and avoids partition key proliferation in paths. |
| **Snapshot** | An OTF concept representing the state of a table at a point in time. Enables time travel queries. Snapshots are retained for a configurable period before expiry. |
| **Path derivation function** | The deterministic algorithm that computes the correct object store path for a given logical object, using the logical container name and object name as inputs. |
| **Bucket policy** | An access control policy attached to a bucket that governs which principals can perform which operations on which path prefixes. Complements but does not replace the logical access model defined in the Object Placement Standard. |

---

## Agent Consumption Instructions

When generating any physically-stored object (typically a table or materialised view):

1. **Locate** a conforming implementation of both this standard and the Object Placement Standard. Both must be present.
2. **Derive the logical container name** using the Object Placement Standard's Derivation Function.
3. **Derive the physical path** using this standard's Path Derivation Function, with the logical container name and object name as inputs.
4. **Generate the DDL** with both the logical container name and the physical path — both are required for OTF table creation.
5. **Apply the file format and partition strategy** declared in Section 5 and Section 6.
6. **Provision physical access** — bucket policies or equivalent — as declared in Section 7, in addition to the logical access model from the Object Placement Standard.
7. **Run both validation procedures** — the Object Placement Standard's Section 8 (logical layer) and this standard's Section 8 (physical layer).
8. **Never invent a path** that is not derived from the declared Path Derivation Function. Ad-hoc paths break deterministic navigation and are a storage governance defect.

**Priority order for locating an implementation:**

1. An explicit file path provided in the current conversation or project instructions
2. `/mnt/skills/user/physical-storage/SKILL.md`
3. A file matching `Physical_Storage_Standard*.md` in the project context
4. **If none of the above exist and object storage is declared in the Object Placement Standard:** STOP. Do not generate any OTF table DDL. Ask the user:

   > *"The Object Placement Standard indicates object storage is in use, but I cannot find a Physical Storage Standard. Before I generate any table DDL with a storage location, I need the physical path convention, file format, and partition strategy. Please provide your Physical Storage Standard."*

---

## Required Sections

Every conforming implementation **MUST** include all eight of the following sections, using these exact headings. An agent may reject an implementation that omits any required section.

---

### Section 1 — Storage Platform Declaration

**What it must contain:**

- The object store service (e.g. Amazon S3, ADLS Gen2, GCS) and any relevant region or configuration constraints
- The Open Table Format in use (e.g. Apache Iceberg, Delta Lake, Apache Hudi) or a statement that no OTF is used and raw files are the storage model
- The version of the OTF if version-specific behaviour applies
- The logical platform this standard extends (must reference the companion Object Placement Standard implementation by name)
- Whether this standard governs all physically-stored objects or a declared subset
- Any objects explicitly excluded from object storage (e.g. temporary tables remain on block storage)

**Why:** Agents must know the storage service, the OTF, and the scope of this standard before generating any DDL with a physical location clause.

---

### Section 2 — Path Model

**What it must contain:**

- The relationship between logical container names (from the Object Placement Standard) and physical path structure — specifically, which logical name segments map to which path segments and in which order
- The bucket strategy: one bucket for all environments, one bucket per environment tier, one bucket per security classification, or another scheme — with rationale
- The root prefix within the bucket (if any) — e.g. `/data/` as a root under which all table paths sit
- Whether path segments use raw values (e.g. `A/D01/COR`) or annotated values (e.g. `prog=A/phase=D01/fn=COR`) — and which annotation style, if any
- Whether views have physical paths (typically they do not — views are logical constructs only)
- The handling of production paths vs development paths — whether the bucket itself provides environment separation or whether the path provides it

**Why:** The path model is the coupling point between the logical standard and the physical standard. Without it, agents cannot derive paths deterministically from logical names, and physical storage becomes ungoverned.

---

### Section 3 — Path Derivation Pattern

**What it must contain:**

- The complete path pattern expressed as an ordered sequence of segments, using `{{SegmentName}}` notation consistent with the Object Placement Standard's naming segments
- For each path segment:
  - The logical name segment it corresponds to (or "bucket-level" if encoded in the bucket name)
  - Its format in the path (raw value, `key=value` annotation, or other)
  - Whether it is mandatory or conditional (e.g. absent in production paths)
- The separator character used between segments (forward slash `/` is conventional; implementations must state their choice)
- The trailing slash convention (include or omit on the table root prefix)
- At least three worked examples covering different lifecycle phases

**The path derivation function signature must accept at minimum:**

```
derive_path(
  logical_container_name,  // the output of the Object Placement Standard's derive_container()
  object_name,             // the logical object name
  bucket                   // the target bucket (may be derived from environment inputs)
) → fully_qualified_object_store_path
```

**Why:** Path derivation must be as deterministic as container name derivation. An agent given a logical container name and object name must be able to compute the physical path without any external lookup.

---

### Section 4 — File Format and Encoding

**What it must contain:**

- The default file format for persistent tables (e.g. Apache Parquet, Apache ORC)
- Any permitted alternative formats and the conditions under which they apply
- The default compression codec and any permitted alternatives
- The schema encoding rules: how column names, data types, and nullability are represented in the file format
- The schema evolution policy: what changes are permitted (add nullable column, rename, widen type) and which require a table drop and recreate
- Any explicit prohibitions (e.g. CSV is never used for persistent tables; JSON is only used for staging)

**Why:** File format and encoding are fixed at table creation time for most OTF implementations. An agent generating a CREATE TABLE statement must apply the correct format. Inconsistent formats across tables in the same logical layer produce heterogeneous storage that is expensive to maintain and migrate.

---

### Section 5 — Partition Strategy

**What it must contain:**

- The partitioning model in use: Hive-style, hidden (OTF-managed), none, or a combination
- If hidden partitioning: the partition spec pattern (e.g. partition by month on `created_date`) — which columns are used as partition keys and by what transform (identity, year, month, day, hour, bucket, truncate)
- If Hive-style: how partition key=value pairs appear in the path and whether they are included in the Path Derivation Pattern from Section 3
- The rule for selecting partition keys: what criteria determine whether a column is a partition key (e.g. high-cardinality columns are excluded; time-based columns are preferred)
- The maximum number of partition columns and the rationale
- How partition evolution is handled when the partition spec changes for an existing table

**Why:** Partition strategy directly affects query performance, file counts, and path structure. An agent generating table DDL must apply the correct partition spec. An incorrect partition (or none at all) can make tables unusable at scale.

---

### Section 6 — Retention and Lifecycle

**What it must contain:**

- The default data retention period for each logical layer (e.g. staging: 7 days; core model: indefinite; temporary: 24 hours)
- The OTF snapshot retention period — how long historical snapshots are retained before expiry (relevant for time travel queries)
- The orphan file cleanup policy — how frequently unreferenced files are removed
- The object store lifecycle rule (if any) — automatic tiering to cheaper storage classes after a defined period
- The workstream retirement procedure at the physical layer: what happens to the object store paths when a workstream is decommissioned (delete, archive, or transfer)
- Who or what is responsible for executing lifecycle actions (a scheduled agent, a human DBA, an automated pipeline)

**Why:** Object store costs accumulate with data volume and snapshot count. Unmanaged retention produces unbounded storage costs, orphaned files, and compliance risks. Agents provisioning tables must understand the lifecycle rules that govern what they create.

---

### Section 7 — Access Model

**What it must contain:**

- The physical access model: bucket policies, IAM roles, access control lists, or a combination — and how they relate to the logical access model defined in the companion Object Placement Standard
- The mapping between logical access principals (roles from the Object Placement Standard) and physical access principals (IAM roles, service accounts, managed identities)
- The principle governing the relationship: physical access must never exceed logical access. A principal with SELECT on a logical view container must not have direct object store read access to the underlying table's physical path.
- The bucket policy structure: which path prefixes map to which access permissions, and how security classification segments in the path are used to enforce tiered access
- Any access controls that apply at the bucket level (e.g. production bucket requires MFA delete; development bucket allows broader read access)
- The handling of cross-service access: if external tools (Spark, Athena, dbt, Glue) access the object store directly, how their access is governed and audited

**Why:** Object store access bypasses the logical access model entirely. A principal with IAM read access to an S3 prefix can read the raw Parquet files without going through any view layer or role-based grant. The physical access model must mirror and reinforce the logical model — not create a bypass around it.

---

### Section 8 — Validation Procedure

**What it must contain:**

- A platform-specific procedure, query, or checklist that confirms correct physical placement after generation
- It must be executable by an agent without human intervention
- It must check at minimum:
  1. The object store path exists and matches the output of the Path Derivation Function for the logical container and object name
  2. The file format of data files matches the declared default (Section 4)
  3. The partition spec of the table matches the declared partition strategy (Section 5)
  4. No data files exist at paths that do not conform to the declared path pattern (orphan or misplaced files)
  5. Physical access principals are correctly scoped — no IAM principal has broader physical access than their logical access model permits
- The expected output of a passing validation
- The expected output of a failing validation and the action an agent must take: halt, report, await instruction — never auto-correct silently

**Why:** Physical storage errors are silent. A table created at the wrong path will appear to work until the path convention matters — for backup, for lifecycle management, for external tool access, or for decommissioning. By then, data may have accumulated in the wrong location and remediation is expensive.

---

## Optional Sections

Implementations MAY include any of the following. Agents should read them if present.

| Section | Purpose |
|---|---|
| **External catalogue integration** | How the OTF table metadata is registered in an external catalogue (AWS Glue, Hive Metastore, Unity Catalog, etc.) and what the agent must do to keep the catalogue consistent with physical storage. |
| **Cross-platform access** | Rules governing access to OTF tables by external compute engines (Spark, Trino, Athena, dbt) — path conventions, credential management, and audit requirements. |
| **Disaster recovery** | Cross-region replication, backup strategy, and recovery time objectives for object store data. |
| **Cost allocation** | How storage costs are attributed to programmes, workstreams, or business units using path-based tagging. |
| **Migration procedure** | How existing BFS (block file storage) tables are migrated to OFS without disruption to consumers. |

---

## Conformance Checklist

An implementation is conforming if it satisfies all of the following:

- [ ] Section 1 (Storage Platform Declaration) names the object store service and OTF in use
- [ ] Section 2 (Path Model) declares the bucket strategy and the relationship between logical names and physical paths
- [ ] Section 3 (Path Derivation Pattern) provides a deterministic function that produces a unique path for any valid logical name and object name combination
- [ ] Section 4 (File Format and Encoding) declares the default format, compression, and schema evolution policy
- [ ] Section 5 (Partition Strategy) declares the partitioning model and partition key selection rules
- [ ] Section 6 (Retention and Lifecycle) declares retention periods for each logical layer and the workstream retirement procedure
- [ ] Section 7 (Access Model) maps logical access principals to physical access principals and states the non-bypass principle
- [ ] Section 8 (Validation Procedure) is executable by an agent and covers all five minimum checks
- [ ] All section headings match the required headings exactly
- [ ] The companion Object Placement Standard implementation is named in Section 1

---

## Non-Goals

This standard does not prescribe:

- The Object Placement Standard's concerns (logical container naming, object naming, logical access model)
- Which object store service to use
- Which Open Table Format to use
- Compute engine choice (Spark, Trino, Teradata, etc.)
- Data modelling methodology (Data Vault, dimensional, etc.)
- How data arrives in the object store (streaming, batch, CDC)
- Query performance optimisation beyond partition strategy

---

## Versioning

Implementations should declare which version of this spec they conform to, using the `Spec version` field in their header. This standard versions independently of the Object Placement Standard. Breaking changes increment the major version; additive changes increment the minor version.
