# AAP 2.6 Migration Automation

Ansible playbooks for migrating Red Hat Ansible Automation Platform from RPM-based installations to containerized or OpenShift deployments, following the official [AAP 2.6 Migration Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/ansible_automation_platform_migration/index).

## Migration Paths

| Playbook | Source | Target |
|----------|--------|--------|
| `migrate_rpm_to_containerized.yml` | AAP 2.6 RPM | AAP 2.6 Containerized |
| `migrate_rpm_to_openshift.yml` | AAP 2.6 RPM | AAP 2.6 on OpenShift |

## Prerequisites

### Control Node

- Ansible 2.15+
- Required collections (install with `ansible-galaxy collection install -r collections/requirements.yml`):
  - `community.postgresql` >= 3.0.0
  - `kubernetes.core` >= 3.0.0 (OpenShift path only)
  - `ansible.posix` >= 1.5.0
  - `community.general` >= 8.0.0

### Source Environment

- AAP 2.6 RPM-based installation
- PostgreSQL 15 (on-premise or external RDS)
- SSH access with sudo privileges to all source hosts

### Target Environment (Containerized)

- AAP 2.6 containerized deployment already installed via the containerized installer
- PostgreSQL 15 container running
- SSH access to target hosts

### Target Environment (OpenShift)

- OpenShift cluster with AAP Operator deployed
- AAP instance created via the Operator (AnsibleAutomationPlatform CR)
- PostgreSQL StatefulSet running
- `oc` CLI installed and `kubeconfig` configured on the bastion host
- Sufficient PVC storage (default 200Gi temporary PVC for migration)

### Source Environment (External Database)

When using an external PostgreSQL database as the source (`source_db_type: external`):

- PostgreSQL 15 instance accessible over the network (e.g., RDS, Azure Database)
- Admin credentials with permission to connect and dump databases
- `psql` and `pg_dump` client tools installed on the source gateway host
- Network connectivity from the source gateway host to the external database

### Target Environment (External Database)

When using an external PostgreSQL database (`target_db_type: external`) with either migration path:

- PostgreSQL 15 instance accessible over the network
- Admin credentials with permission to create/drop databases and alter user roles
- `psql` and `pg_restore` client tools installed on the restore host:
  - Containerized path: the target gateway host (default) or a custom `db_restore_host`
  - OpenShift path: the bastion host (default), or set `db_restore_host: pod` to use a temporary in-cluster pod
- Network connectivity from the restore host to the external database

## Quick Start

### 1. Install Collections

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### 2. Configure the Inventory

Copy and edit the appropriate inventory for your migration path:

**RPM to Containerized:**
```bash
cp -r inventories/rpm_to_containerized inventories/my_migration
vi inventories/my_migration/hosts.yml
```

**RPM to OpenShift:**
```bash
cp -r inventories/rpm_to_openshift inventories/my_migration
vi inventories/my_migration/hosts.yml
```

**With External Target Database (either path):**
```bash
cp -r inventories/rpm_to_containerized_external_db inventories/my_migration
# or: cp -r inventories/rpm_to_openshift_external_db inventories/my_migration
vi inventories/my_migration/hosts.yml
vi inventories/my_migration/group_vars/all.yml  # Set target_pg_host, target_pg_port, etc.
```

**With External Source Database (either path):**
```bash
cp -r inventories/rpm_to_containerized_external_source_db inventories/my_migration
# or: cp -r inventories/rpm_to_openshift_external_source_db inventories/my_migration
vi inventories/my_migration/hosts.yml
vi inventories/my_migration/group_vars/all.yml  # Set source_pg_host, source_pg_port, etc.
vi inventories/my_migration/group_vars/source/main.yml  # Set component DB credentials
```

**With Both Source and Target External Databases (either path):**
```bash
cp -r inventories/rpm_to_containerized_external_source_and_target_db inventories/my_migration
# or: cp -r inventories/rpm_to_openshift_external_source_and_target_db inventories/my_migration
vi inventories/my_migration/hosts.yml
vi inventories/my_migration/group_vars/all.yml  # Set both source and target DB settings
vi inventories/my_migration/group_vars/source.yml  # Set source component DB credentials
```

Update the following in your inventory:

- **`hosts.yml`** -- Replace placeholder hostnames with actual FQDNs for all source and target hosts.
- **`group_vars/all.yml`** -- Review component toggles (`migrate_controller`, `migrate_hub`, `migrate_gateway`, `migrate_eda`) and adjust artifact paths if needed.
- **`group_vars/source/main.yml`** -- Verify secret key file paths match your source installation.
- **`group_vars/target/main.yml`** -- Set target-specific paths (installer inventory, kubeconfig, namespace, etc.).
- **`group_vars/vault.yml`** -- Populate database credentials (see [Vault Variables](#vault-variables) below).

### 3. Encrypt Sensitive Variables

```bash
ansible-vault encrypt inventories/my_migration/group_vars/vault.yml
```

### 4. Run the Migration

**Full migration (all phases):**
```bash
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml --ask-vault-pass
```

**Or for OpenShift:**
```bash
ansible-playbook -i inventories/my_migration migrate_rpm_to_openshift.yml --ask-vault-pass
```

## Migration Phases

The playbooks execute phases 0 through 7 sequentially. Each phase can be run independently using tags.

| Phase | Tag | Description | Role(s) |
|-------|-----|-------------|---------|
| 0 | `validate` | Validate prerequisites, connectivity, and versions | `common` |
| 1 | `assess` | Assess source RPM environment (AAP & PostgreSQL versions) | `source_assess` |
| 2-3 | `export` | Export databases, secrets, custom configs, and package artifact | `source_export` |
| 4 | `assess` | Assess target environment readiness | `target_*_assess` |
| 5 | `import` | Stop services, restore databases, start services | `target_*_import` |
| 6 | `reconcile` | Reconcile gateway, secrets, instances, and hub content | `target_*_reconcile` |
| 7 | `validate` | Health checks and post-migration validation | `validate` |

### Running Individual Phases

```bash
# Assessment only (phases 0, 1, 4)
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml --tags assess

# Export only (phases 2-3)
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml --tags export

# Import only (phase 5)
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml --tags import

# Reconcile only (phase 6)
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml --tags reconcile

# Validate only (phases 0 and 7)
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml --tags validate
```

## Inventory Structure

```
inventories/
  rpm_to_containerized/
    hosts.yml                  # Host definitions and group structure
    group_vars/
      all.yml                  # Shared variables (versions, toggles, paths)
      source.yml               # Source host settings (secret paths, config dirs)
      target.yml               # Target host settings (services, installer paths)
      vault.yml                # Sensitive credentials (encrypt with ansible-vault)
  rpm_to_openshift/
    hosts.yml                  # Host definitions (includes OCP bastion)
    group_vars/
      all.yml                  # Shared variables + OCP-specific defaults
      source.yml               # Source host settings
      target.yml               # OCP settings (namespace, kubeconfig, PVC size)
      vault.yml                # Sensitive credentials
  rpm_to_containerized_external_db/
    hosts.yml                  # No target_db group (external database)
    group_vars/
      all.yml                  # Includes target_db_type: external and connection vars
      source.yml               # Source host settings
      target.yml               # Target host settings (same as managed)
      vault.yml                # Sensitive credentials including external DB password
  rpm_to_openshift_external_db/
    hosts.yml                  # Same as OpenShift (source + bastion)
    group_vars/
      all.yml                  # Includes target_db_type: external, gateway_hostname
      source.yml               # Source host settings
      target.yml               # OCP settings (same as managed)
      vault.yml                # Sensitive credentials including external DB password
  rpm_to_containerized_external_source_db/
    hosts.yml                  # No source_db group (external source database)
    group_vars/
      all.yml                  # Includes source_db_type: external and connection vars
      source.yml               # Source database connection settings per component
      target.yml               # Target host settings (managed DB)
      vault.yml                # Sensitive credentials including source DB passwords
  rpm_to_openshift_external_source_db/
    hosts.yml                  # No source_db group (external source database)
    group_vars/
      all.yml                  # Includes source_db_type: external and connection vars
      source.yml               # Source database connection settings per component
      target_ocp_bastion.yml   # OCP bastion settings
      vault.yml                # Sensitive credentials including source DB passwords
  rpm_to_containerized_external_source_and_target_db/
    hosts.yml                  # No source_db or target_db groups (both external)
    group_vars/
      all.yml                  # Includes both source_db_type and target_db_type: external
      source.yml               # Source database connection settings per component
      target.yml               # Target host settings
      vault.yml                # Sensitive credentials for both source and target DBs
  rpm_to_openshift_external_source_and_target_db/
    hosts.yml                  # No source_db group (both databases external)
    group_vars/
      all.yml                  # Includes both source_db_type and target_db_type: external
      source.yml               # Source database connection settings per component
      target_ocp_bastion.yml   # OCP bastion settings
      vault.yml                # Sensitive credentials for both source and target DBs
```

### Key Variables in `all.yml`

| Variable | Default | Description |
|----------|---------|-------------|
| `required_aap_version` | `"2.6"` | Expected AAP version on source |
| `required_postgresql_version` | `"15"` | Expected PostgreSQL version |
| `confirm_destructive_operations` | `true` | Pause for confirmation before destructive actions |
| `migrate_controller` | `true` | Include Controller in migration |
| `migrate_hub` | `true` | Include Automation Hub in migration |
| `migrate_gateway` | `true` | Include Gateway in migration |
| `migrate_eda` | `false` | Include Event-Driven Ansible in migration |
| `artifact_dir` | `/tmp/backups/artifact` | Working directory for export artifact |
| `artifact_archive` | `/tmp/backups/artifact.tar` | Final packaged artifact path |
| `db_dump_timeout` | `3600` | Database dump timeout in seconds |
| `db_restore_timeout` | `3600` | Database restore timeout in seconds |
| `source_db_type` | `managed` | `managed` or `external` — whether source DB is on-premise or external (RDS, etc.) |
| `source_pg_host` | — | External source database hostname (required when `source_db_type: external`) |
| `source_pg_port` | `5432` | External source database port |
| `source_pg_ssl_mode` | `prefer` | SSL mode for external source DB connections |
| `source_pg_admin_user` | — | Admin username for external source database |
| `target_db_type` | `managed` | `managed` or `external` — whether target DB is managed by installer/operator |
| `target_pg_host` | — | External target database hostname (required when `target_db_type: external`) |
| `target_pg_port` | `5432` | External target database port |
| `target_pg_ssl_mode` | `prefer` | SSL mode for external target DB connections |
| `db_restore_host` | auto | Host to run `psql`/`pg_restore` from; set to `pod` for OpenShift in-cluster restore |

### Vault Variables

The `vault.yml` file should contain database credentials:

**For managed source and target databases:**
```yaml
gateway_admin_password: <your-password>
```

**For external source database:**
```yaml
source_pg_admin_user: <your-db-admin-username>
source_pg_admin_password: <source-db-admin-password>
controller_pg_password: <controller-db-password>
hub_pg_password: <hub-db-password>
gateway_pg_password: <gateway-db-password>
```

**For external target database:**
```yaml
target_pg_admin_user: <your-db-admin-username>
target_pg_admin_password: <target-db-admin-password>
gateway_admin_password: <gateway-admin-password>
```

**For both external source and target databases:**
```yaml
source_pg_admin_user: <source-db-admin-username>
source_pg_admin_password: <source-db-admin-password>
controller_pg_password: <controller-db-password>
hub_pg_password: <hub-db-password>
gateway_pg_password: <gateway-db-password>
target_pg_admin_user: <target-db-admin-username>
target_pg_admin_password: <target-db-admin-password>
gateway_admin_password: <gateway-admin-password>
```

### OpenShift-Specific Variables in `target.yml`

| Variable | Default | Description |
|----------|---------|-------------|
| `kubeconfig` | `~/.kube/config` | Path to kubeconfig file |
| `ocp_namespace` | `aap` | OpenShift namespace for AAP |
| `aap_instance_name` | `aap` | Name of the AnsibleAutomationPlatform CR |
| `postgres_statefulset` | `aap-postgres-15` | PostgreSQL StatefulSet name |
| `temp_pvc_size` | `200Gi` | Temporary PVC size for artifact transfer |

## Role Reference

```
roles/
  common/                           # Phase 0: Prerequisite validation
  source_assess/                    # Phase 1: Source environment assessment
  source_export/                    # Phases 2-3: Database dump, secrets & config export
  target_containerized_assess/      # Phase 4: Containerized target assessment
  target_containerized_import/      # Phase 5: Database restore (containerized)
  target_containerized_reconcile/   # Phase 6: Gateway, secrets, instance reconciliation
  target_openshift_assess/          # Phase 4: OpenShift target assessment
  target_openshift_import/          # Phase 5: Database restore (OpenShift)
  target_openshift_reconcile/       # Phase 6: Gateway, secrets, instance reconciliation
  validate/                         # Phase 7: Post-migration health checks
```

## Safety Features

- **Confirmation prompts**: Destructive operations (database dumps, restores, service stops, installer re-runs) pause for manual confirmation when `confirm_destructive_operations: true`.
- **Checksum validation**: SHA256 checksums are generated and verified for all database dumps and the packaged artifact.
- **Pre-import backup**: A backup of the target containerized environment is created before the import phase begins.
- **Component toggles**: Migrate only the components you need by setting `migrate_controller`, `migrate_hub`, `migrate_gateway`, and `migrate_eda`.
- **Version enforcement**: Playbooks validate that both AAP and PostgreSQL meet version requirements before proceeding.
- **Configurable timeouts**: Database dump and restore operations use `db_dump_timeout` and `db_restore_timeout` (default 1 hour each) to handle large databases.

## Post-Migration Manual Steps

After the playbooks complete, verify the following manually:

1. **Projects, inventories, and job templates** are present and correctly configured
2. **Collections and namespaces** are available in Automation Hub
3. **Instance groups** are reconciled for the new environment
4. **Decision environments and execution environments** are updated
5. **Credentials** are valid (especially machine credentials, cloud credentials, SCM credentials)
6. **RBAC rules** reflect the new deployment topology
7. **Job execution** works end-to-end (run test jobs with various credential types)
8. **Workflow templates** execute correctly
9. **SSO integration** functions if configured
10. **Content synchronization** from external sources works
11. **API tokens** are valid

## Troubleshooting

### Resuming After Failure

The playbooks are designed to be re-runnable. If a phase fails, fix the issue and re-run the playbook. Use tags to skip completed phases:

```bash
# Resume from the import phase
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml --tags import,reconcile,validate
```

### Increasing Timeouts for Large Databases

If database dump or restore operations time out, increase the timeout values in `group_vars/all.yml`:

```yaml
db_dump_timeout: 7200    # 2 hours
db_restore_timeout: 7200  # 2 hours
```

### Skipping Confirmation Prompts

For unattended runs (use with caution):

```yaml
# In group_vars/all.yml
confirm_destructive_operations: false
```

### Verbose Output

```bash
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml -vvv
```
