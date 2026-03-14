# External Database Support for AAP Migration

## Problem

The migration tooling assumes the target PostgreSQL database is deployed and managed by the AAP installer (containerized path) or operator (OpenShift path). Users who run AAP against an external database — Amazon RDS, Azure Database for PostgreSQL, a DBA-managed on-prem instance, or any PostgreSQL server not managed by the AAP lifecycle tooling — cannot use the migration playbooks today without manual workarounds.

## Goals

- Support external (non-installer/operator-managed) PostgreSQL as a target for both migration paths.
- Preserve existing managed-database behavior as the default with zero breaking changes.
- Keep the change surface minimal: branch inside existing roles via conditionals rather than duplicating roles or playbooks.

## Non-Goals

- Provisioning or configuring the external database itself (users bring a running PostgreSQL instance).
- Supporting non-PostgreSQL databases.
- Changing how the source-side export works (source database handling is unaffected).

---

## Variable Model

### New Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `target_db_type` | `managed` \| `external` | `managed` | Whether the target database is managed by the AAP installer/operator or is external. |
| `target_pg_port` | integer | `5432` | PostgreSQL port on the target database host. |
| `target_pg_ssl_mode` | string | `prefer` | SSL mode for PostgreSQL connections (`disable`, `allow`, `prefer`, `require`, `verify-ca`, `verify-full`). Passed as `PGSSLMODE` environment variable to `psql`/`pg_restore`. |
| `db_restore_host` | string | see below | Host from which `psql`/`pg_restore` commands are executed. Containerized default: `groups['target_gateway'][0]`. OpenShift default: `groups['target_ocp_bastion'][0]`. Set to the literal string `"pod"` on OpenShift to use a temporary in-cluster pod instead. Either `db_restore_host` or the relevant inventory group must be populated. |
| `postgres_client_image` | string | `registry.redhat.io/rhel9/postgresql-15:latest` | Container image with PostgreSQL client tools. Used only for OpenShift external + pod restore when there is no StatefulSet to derive the image from. |

### Existing Variables (Promoted to Required When External)

| Variable | Description |
|----------|-------------|
| `target_pg_host` | External database hostname. Already exists as an optional override; becomes required when `target_db_type: external`. |
| `target_pg_admin_user` | Admin user for restore operations. Already exists; becomes required when external. |
| `target_pg_admin_password` | Admin password (should be in vault). Already exists; becomes required when external. |

### Unchanged Variables

Per-component credentials (`controller_pg_user`, `hub_pg_user`, `gateway_pg_user` and their `_database` counterparts) work identically in both modes.

---

## Containerized Path

### Phase 4: `target_containerized_assess`

**Managed mode** — unchanged:
- Runs `podman exec postgresql psql` on `target_db` to check PostgreSQL version.
- Creates backup via `ansible.containerized_installer.backup`.

**External mode:**
- Validates that `psql` and `pg_restore` client tools are installed on `db_restore_host` (default: `target_gateway`).
- Connects to `target_pg_host:target_pg_port` with `target_pg_admin_user` / `target_pg_admin_password` to verify connectivity.
- Runs `SHOW server_version` over the network connection to check PostgreSQL version.
- Skips containerized installer backup (no local database to back up).

### Phase 5: `target_containerized_import`

**`stop_services.yml` / `start_services.yml`** — unchanged in both modes. These manage AAP application services, not the database.

**`restore_databases.yml`:**

Managed mode — unchanged:
- Delegates to `groups['target_db'][0]`.
- Uses `target_pg_host | default(groups['target_db'][0])`.

External mode:
- Delegates to `db_restore_host` (default: first host in `target_gateway`).
- Connects to `target_pg_host:target_pg_port` with explicit credentials.
- `community.postgresql.postgresql_user` calls for CREATEDB grant/revoke use the same external connection parameters.
- The `pg_restore` command is structurally identical — only the host, port, and delegation target change.

### Phase 6: `target_containerized_reconcile`

Unchanged in both modes. Runs `aap-gateway-manage migrate` and clears gateway data; no direct database connection changes needed.

---

## OpenShift Path

### Phase 4: `target_openshift_assess`

**Managed mode** — unchanged:
- Verifies PostgreSQL StatefulSet exists in the cluster.

**External mode:**
- Skips StatefulSet verification.
- Determines restore execution location from `db_restore_host`:
  - **Bastion** (default, resolves to `groups['target_ocp_bastion'][0]`): verifies `psql`/`pg_restore` client tools on the bastion, tests connectivity to `target_pg_host:target_pg_port`.
  - **Pod** (`db_restore_host: pod` — literal string `"pod"`): defers connectivity check to import phase when the temp pod is created.
- Still verifies `oc`, kubeconfig, namespace, and AAP instance (needed regardless of database type).

### Phase 5: `target_openshift_import`

The import flow in `main.yml` includes tasks conditionally based on `target_db_type` and `db_restore_host`:

```yaml
# main.yml flow (simplified)
- include_tasks: idle_aap.yml                    # always
- include_tasks: scale_operators.yml             # always (internal branching)
- include_tasks: extract_artifact_on_bastion.yml # only when external + bastion
  when: target_db_type == 'external' and db_restore_host != 'pod'
- include_tasks: create_temp_resources.yml       # skip when external + bastion
  when: target_db_type != 'external' or db_restore_host == 'pod'
- include_tasks: transfer_artifact.yml           # skip when external + bastion
  when: target_db_type != 'external' or db_restore_host == 'pod'
- include_tasks: restore_databases.yml           # always (internal branching)
- include_tasks: update_secrets.yml              # always (internal branching)
- include_tasks: cleanup_temp.yml                # skip when external + bastion
  when: target_db_type != 'external' or db_restore_host == 'pod'
- include_tasks: scale_up.yml                    # always
```

**Artifact extraction (external + bastion only):** When `target_db_type == 'external'` and `db_restore_host != 'pod'`, the artifact must be extracted on the bastion before restore and secret update tasks can run. A new task file `extract_artifact_on_bastion.yml` is included conditionally in `main.yml` (in the slot where `create_temp_resources` and `transfer_artifact` would normally run). It extracts `artifact_archive` to `artifact_dir` on the bastion host, mirroring what `transfer_artifact.yml` does inside the pod for the managed path. Users must ensure `artifact_archive` has been copied to the bastion before running the playbook (same as the existing requirement to have the artifact on `target_gateway` for containerized).

**`idle_aap.yml`** — unchanged.

**`scale_operators.yml`:**
- Managed mode — unchanged: scales down operators and PostgreSQL StatefulSet.
- External mode: scales down operators only. Skips StatefulSet scaling (no managed StatefulSet).

**`create_temp_resources.yml`:**
- Managed mode — unchanged: gets PostgreSQL image from StatefulSet pod, creates temp PVC + deployment.
- External mode, bastion restore: skipped entirely (conditional on `main.yml`).
- External mode, pod restore: cannot derive `postgres_image` from the StatefulSet (it doesn't exist). Uses new variable `postgres_client_image` (user-provided or defaulting to `registry.redhat.io/rhel9/postgresql-15:latest`) to create the temp pod. The pod only needs `psql`/`pg_restore` client tools — it does not run a PostgreSQL server.

**`transfer_artifact.yml`:**
- Managed mode — unchanged: copies artifact into the temp pod.
- External mode, bastion restore: skipped entirely.
- External mode, pod restore: still runs — copies artifact into the temp pod for restore.

**`restore_databases.yml` and `restore_single_db.yml`:**
- Managed mode — unchanged: reads credentials from OCP secrets, uses `oc exec` into temp pod to run `psql`/`pg_restore` against the StatefulSet.
- External mode (both pod and bastion): `restore_databases.yml` skips the "Get PostgreSQL passwords from OCP secrets" and "Parse database credentials" tasks entirely (`when: target_db_type != 'external'`). Instead, it builds the credential structure from inventory variables: `target_pg_host`, `target_pg_port`, `target_pg_admin_user`, `target_pg_admin_password`, plus per-component `controller_pg_user`/`hub_pg_user`/`gateway_pg_user` and their `_password` counterparts from vault.
- External mode, pod restore: `restore_single_db.yml` uses `oc exec` into the temp pod but connects to `target_pg_host:target_pg_port` instead of `postgres_statefulset`. `PGSSLMODE` is set to `target_pg_ssl_mode`.
- External mode, bastion restore: instead of including `restore_single_db.yml`, includes a new `restore_single_db_external.yml` that runs `psql`/`pg_restore` directly on the bastion host (via `delegate_to: db_restore_host`) against `target_pg_host:target_pg_port`. `PGSSLMODE` and `PGPASSWORD` are set in the `environment` block.

**`update_secrets.yml`:**
- Managed mode — unchanged: reads `secrets.yml` from the temp pod via `k8s_exec`.
- External mode, pod restore: unchanged — reads from the temp pod (artifact was transferred there).
- External mode, bastion restore: reads `secrets.yml` from the artifact on the bastion (`{{ artifact_dir }}/secrets.yml`) via `ansible.builtin.slurp` delegated to the bastion host. The rest of the task (updating OCP secrets via `oc set data`) is unchanged — it always runs on the bastion.

**`cleanup_temp.yml`:** skipped when no temp resources were created (external + bastion restore path).

**`scale_up.yml`** — unchanged: scales operators back up, un-idles AAP.

### Phase 6: `target_openshift_reconcile`

Unchanged in both modes. Operates on gateway pods and APIs; no direct database connection changes.

---

## Validation

### Phase 0: `common` Role

Add validation when `target_db_type: external`:
- Assert `target_pg_host`, `target_pg_admin_user`, and `target_pg_admin_password` are defined.
- Fail (not just warn) if `target_db_type: external` and `target_db` inventory group has hosts — this is an invalid configuration.
- Warn if `target_db_type: managed` but `target_db` group is empty (containerized path only).
- Assert that either `db_restore_host` is set or the relevant inventory group (`target_gateway` for containerized, `target_ocp_bastion` for OpenShift) is populated.
- Assert `target_pg_admin_password` is defined (should reference a vault variable, e.g. `target_pg_admin_password: "{{ vault_target_pg_admin_password }}"`; the README will document this pattern).

### Phase 7: `validate` Role

- Add external database connectivity check: run `psql -h {{ target_pg_host }} -p {{ target_pg_port }} -U {{ target_pg_admin_user }} -c 'SELECT 1'` from `db_restore_host` to verify the external database is reachable post-migration.
- Existing health checks (API endpoints, login tests) remain unchanged.
- Note: the `validate` role uses `gateway_hostname | default(groups['target_gateway'][0])`. OpenShift inventories do not define `target_gateway`, so OpenShift users (both managed and external) must set `gateway_hostname` explicitly. This is pre-existing behavior; the README update should call it out for external DB users.

---

## Inventory Examples

Two new example inventories alongside the existing ones:

### `inventories/rpm_to_containerized_external_db/`

```yaml
# hosts.yml
all:
  children:
    source:
      children:
        source_controller:
          hosts:
            source-controller.example.com:
        source_hub:
          hosts:
            source-hub.example.com:
        source_gateway:
          hosts:
            source-gateway.example.com:
        source_db:
          hosts:
            source-db.example.com:
    target:
      children:
        target_controller:
          hosts:
            target-controller.example.com:
        target_hub:
          hosts:
            target-hub.example.com:
        target_gateway:
          hosts:
            target-gateway.example.com:
        target_eda:
          hosts:
            target-eda.example.com:
        # No target_db group — database is external
```

```yaml
# group_vars/all.yml
target_db_type: external
target_pg_host: my-rds-instance.us-east-1.rds.amazonaws.com
target_pg_port: 5432
target_pg_ssl_mode: require
```

### `inventories/rpm_to_openshift_external_db/`

```yaml
# hosts.yml
all:
  children:
    source:
      children:
        source_controller:
          hosts:
            source-controller.example.com:
        source_hub:
          hosts:
            source-hub.example.com:
        source_gateway:
          hosts:
            source-gateway.example.com:
        source_db:
          hosts:
            source-db.example.com:
    target:
      children:
        target_ocp_bastion:
          hosts:
            bastion.example.com:
```

```yaml
# group_vars/all.yml
target_db_type: external
target_pg_host: my-azure-db.postgres.database.azure.com
target_pg_port: 5432
target_pg_ssl_mode: require

# Required for validate role on OpenShift
gateway_hostname: aap.apps.ocp.example.com

# Restore from bastion (default):
# db_restore_host: bastion.example.com

# Or restore from a temp pod in the cluster:
# db_restore_host: pod
```

---

## Files Changed

| File | Change |
|------|--------|
| `inventories/rpm_to_containerized/group_vars/all.yml` | Add `target_db_type: managed` default, `target_pg_port`, `target_pg_ssl_mode` |
| `inventories/rpm_to_openshift/group_vars/all.yml` | Add `target_db_type: managed` default, `target_pg_port`, `target_pg_ssl_mode` |
| `roles/common/tasks/main.yml` | Add external DB variable validation and `db_restore_host` validation |
| `roles/common/defaults/main.yml` | Add `target_db_type`, `target_pg_port`, `target_pg_ssl_mode` defaults |
| `roles/target_containerized_assess/tasks/main.yml` | Add external DB assess path |
| `roles/target_containerized_import/tasks/restore_databases.yml` | Branch delegation and connection params by `target_db_type` |
| `roles/target_containerized_import/defaults/main.yml` | Add `db_restore_host` default |
| `roles/target_openshift_assess/tasks/main.yml` | Add external DB assess path |
| `roles/target_openshift_import/tasks/main.yml` | Add `when` conditionals on `create_temp_resources`, `transfer_artifact`, and `cleanup_temp` includes |
| `roles/target_openshift_import/tasks/scale_operators.yml` | Skip StatefulSet scaling when external |
| `roles/target_openshift_import/tasks/create_temp_resources.yml` | Use `postgres_client_image` when external (no StatefulSet to derive image from) |
| `roles/target_openshift_import/tasks/transfer_artifact.yml` | No internal changes (skipped via `main.yml` conditional when external + bastion) |
| `roles/target_openshift_import/tasks/restore_databases.yml` | Build credentials from inventory vars when external; branch to `restore_single_db_external.yml` for bastion restore |
| `roles/target_openshift_import/tasks/restore_single_db.yml` | Connect to `target_pg_host:target_pg_port` instead of `postgres_statefulset` when external + pod restore |
| `roles/target_openshift_import/tasks/restore_single_db_external.yml` | **New file.** Bastion-based restore: runs `psql`/`pg_restore` directly on the bastion via `delegate_to` |
| `roles/target_openshift_import/tasks/extract_artifact_on_bastion.yml` | **New file.** Extracts `artifact_archive` to `artifact_dir` on the bastion for external + bastion restore |
| `roles/target_openshift_import/tasks/update_secrets.yml` | Add external + bastion path: read `secrets.yml` from artifact on bastion via `ansible.builtin.slurp` instead of `k8s_exec` |
| `roles/target_openshift_import/tasks/cleanup_temp.yml` | No internal changes (skipped via `main.yml` conditional when external + bastion) |
| `roles/target_openshift_import/defaults/main.yml` | Add `db_restore_host` default, `postgres_client_image` default |
| `roles/validate/tasks/main.yml` | Add external DB connectivity check |
| `inventories/rpm_to_containerized_external_db/` | New example inventory (hosts.yml, group_vars/) |
| `inventories/rpm_to_openshift_external_db/` | New example inventory (hosts.yml, group_vars/) |
| `README.md` | Document external database support, new variables, example commands, vault pattern for `target_pg_admin_password`, `gateway_hostname` requirement for OpenShift |

## Files Unchanged

| File | Reason |
|------|--------|
| `migrate_rpm_to_containerized.yml` | Branching happens inside roles |
| `migrate_rpm_to_openshift.yml` | Branching happens inside roles |
| `roles/source_assess/` | Source-side only |
| `roles/source_export/` | Source-side only |
| `roles/target_containerized_reconcile/` | No direct DB connections |
| `roles/target_openshift_reconcile/` | No direct DB connections |
