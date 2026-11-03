# ADR-008: Immutable workflow versions; executions pin a version

- Status: Accepted
- Date: 2026-11-03

## Context

Admins edit workflows in Automation Studio while executions may be running. Executions must be reproducible: "why did this participant receive X?" must be answerable months later.

## Decision

1. `automation_workflows` holds identity and a mutable **draft** (`draft_definition jsonb`).
2. **Publish** validates the draft (Zod schema, graph acyclicity, capability checks, node limits). It then inserts an **immutable** `workflow_versions` row with `version_number`, `definition`, `definition_checksum (sha256)`, `schema_version`, `published_by` and `published_at`, and sets `automation_workflows.active_version_id`.
3. `workflow_versions` rows are never updated. This is enforced by a DB trigger that rejects `UPDATE` on `definition` and `definition_checksum`.
4. Every `workflow_execution` stores `workflow_version_id`. The interpreter loads that exact version for the whole execution.
5. **In-flight executions continue on their pinned version** by default. On publish, the admin can choose to:
   - `continue`: the default
   - `cancel_inflight`: cancel in-flight executions and leave new triggers on the new version
   - The **"migrate in-flight to new version"** option is **not** supported. Mapping step state between graphs is unsafe.
6. **Interpreter compatibility**: the workflow interpreter supports every `schema_version` that still has non-terminal executions. CI checks this by replaying golden fixtures for each supported schema version.
7. **Inngest step ID stability**: step IDs are derived from `nodeId` + iteration index, never from array positions. Renaming a node ID in a new version doesn't affect old executions, because they are pinned.
8. Rollback means publishing a new version copied from an older one. History stays linear.

## Consequences

- ✅ Full reproducibility and auditability.
- ⚠️ Long-running executions on old versions may linger. A dashboard shows "executions on non-active versions".
