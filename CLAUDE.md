# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible playbooks for migrating Red Hat Ansible Automation Platform (AAP) 2.6 from RPM-based installations to containerized or OpenShift deployments. Based on the official AAP 2.6 Migration Guide.

Two migration paths:
- **RPM → Containerized**: `migrate_rpm_to_containerized.yml`
- **RPM → OpenShift**: `migrate_rpm_to_openshift.yml`

## Common Commands

```bash
# Install required collections
ansible-galaxy collection install -r collections/requirements.yml

# Run full migration (containerized)
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml --ask-vault-pass

# Run full migration (OpenShift)
ansible-playbook -i inventories/my_migration migrate_rpm_to_openshift.yml --ask-vault-pass

# Run individual phases via tags: assess, export, import, reconcile, validate
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml --tags export --ask-vault-pass

# Syntax check
ansible-playbook --syntax-check migrate_rpm_to_containerized.yml
ansible-playbook --syntax-check migrate_rpm_to_openshift.yml

# Verbose output for debugging
ansible-playbook -i inventories/my_migration migrate_rpm_to_containerized.yml -vvv --ask-vault-pass
```

## Architecture

### Migration Phases (Sequential)

Both playbooks execute the same 8-phase pipeline. Phases 0-3 run on source hosts; phases 4-7 run on target hosts. The target roles differ by migration path (containerized vs OpenShift).

| Phase | Tag | Role(s) |
|-------|-----|---------|
| 0 | assess | `common` — prerequisite validation |
| 1 | assess | `source_assess` — source RPM environment check |
| 2-3 | export | `source_export` — DB dump, secrets, configs → packaged artifact |
| 4 | assess | `target_containerized_assess` or `target_openshift_assess` |
| 5 | import | `target_containerized_import` or `target_openshift_import` |
| 6 | reconcile | `target_containerized_reconcile` or `target_openshift_reconcile` |
| 7 | validate | `validate` — post-migration health checks |

### Role Structure

Roles follow standard Ansible layout (`defaults/main.yml`, `tasks/main.yml`, `templates/`). The `source_export` role is the most complex, with subtask files for each export step (gather_db_settings, validate_databases, dump_databases, export_secrets, copy_custom_configs, create_manifest, package_artifact).

### Inventory Groups

**Containerized path** uses `source` and `target` groups, each with subgroups: `source_controller`, `source_hub`, `source_gateway`, `source_db`, `target_controller`, `target_hub`, `target_gateway`, `target_db`, `target_eda`.

**OpenShift path** uses `source` (same subgroups) and `target_ocp_bastion` (single bastion host that runs `oc` commands).

**External database configurations**: When using external databases (RDS, Azure Database, etc.), leave the corresponding inventory group empty (`source_db` for external source, `target_db` for external target) and set the appropriate `*_db_type` variable to `external`.

**Inventory variable structure**: Variables are organized in `group_vars` subdirectories:
- `group_vars/all/main.yml` - shared variables for all hosts
- `group_vars/all/vault.yml` - sensitive credentials (ansible-vault encrypted)
- `group_vars/source/main.yml` - source-specific settings
- `group_vars/target/main.yml` - target-specific settings

### Key Variables

- Component toggles: `migrate_controller`, `migrate_hub`, `migrate_gateway`, `migrate_eda`
- Safety: `confirm_destructive_operations` (default true) triggers `ansible.builtin.pause` before destructive actions
- Paths: `artifact_dir`, `artifact_archive` control where export artifacts are staged/packaged
- Timeouts: `db_dump_timeout`, `db_restore_timeout` (default 3600s)
- Source DB: `source_db_type` (`managed`|`external`), `source_pg_host`, `source_pg_port`, `source_pg_ssl_mode`, `source_pg_admin_user`, `source_pg_admin_password`
- Target DB: `target_db_type` (`managed`|`external`), `target_pg_host`, `target_pg_port`, `target_pg_ssl_mode`, `target_pg_admin_user`, `target_pg_admin_password`
- Containerized installer (used during assess backup and reconcile phases) - **MUST be configured**:
  - `containerized_installer_dir`: Path to the AAP containerized installer bundle directory (default: `~/ansible-automation-platform-containerized-setup-bundle-2.6-1`)
  - `containerized_installer_inventory`: Inventory file relative to `containerized_installer_dir` or absolute path (default: `inventory`)
  - `containerized_installer_command`: Command to run the installer, executed from `containerized_installer_dir` (default: `./setup.sh`)
  - `containerized_installer_backup_command`: Command to backup the containerized environment, executed from `containerized_installer_dir` (default: `./setup.sh -b`)
- Credentials in `group_vars/all/vault.yml` (gitignored, must be ansible-vault encrypted)

## Conventions

- All modules use FQCN (e.g., `ansible.builtin.command`, not `command`)
- Task files are decomposed into focused subtask files included from `tasks/main.yml`
- Destructive operations use `ansible.builtin.pause` for confirmation when `confirm_destructive_operations` is true
- SHA256 checksums validate database dumps and the packaged artifact
- The `collections/ansible_collections/` directory is gitignored; only `collections/requirements.yml` is tracked
