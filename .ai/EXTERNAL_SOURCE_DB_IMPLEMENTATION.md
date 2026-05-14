# External Source Database Support Implementation

## Overview

Added support for using external RDS/cloud databases as the source database for AAP migrations. Previously, only external target databases were supported.

## Changes Made

### 1. Core Configuration (`roles/common`)

**`defaults/main.yml`:**
- Added `source_db_type` (default: `managed`)
- Added `source_pg_port` (default: `5432`)
- Added `source_pg_ssl_mode` (default: `prefer`)
- Mirrored the existing target database configuration pattern

**`tasks/main.yml`:**
- Added validation for `source_db_type` value
- Added validation that required variables are defined for external source DB
- Added checks to prevent `source_db` inventory group from having hosts when `source_db_type: external`
- Added warning when `source_db_type: managed` but `source_db` group is empty
- Added display of source external database configuration

### 2. Source Assessment (`roles/source_assess`)

**`tasks/main.yml`:**
- Split PostgreSQL version check into two tasks:
  - Managed DB: Uses `become: postgres` on `source_db[0]` (existing behavior)
  - External DB: Connects via network to `source_pg_host` using admin credentials
- External DB check uses `PGPASSWORD` environment variable for authentication
- Both paths converge to same validation logic

### 3. Source Export (`roles/source_export`)

**`defaults/main.yml`:**
- Updated `components` list to conditionally set `pg_host` based on `source_db_type`
- Added `pg_port` and `pg_password` fields to each component
- Defaults use `source_pg_host` when `source_db_type: external`
- Passwords default to `source_pg_admin_password` for external databases

**`tasks/dump_databases.yml`:**
- Added `-p {{ item.pg_port }}` flag to all `psql` and `pg_dump` commands
- Added `environment` block with `PGPASSWORD` and `PGSSLMODE` to both connectivity check and dump tasks
- Uses `omit` filter to avoid passing empty passwords for managed databases

### 4. Inventory Examples

Created two new inventory examples:

**`inventories/rpm_to_containerized_external_source_db/`:**
- Empty `source_db` group in `hosts.yml`
- `all.yml` sets `source_db_type: external` and defines RDS connection variables
- `source.yml` overrides component DB hosts to use `source_pg_host`
- Includes vault variable references for DB passwords

**`inventories/rpm_to_openshift_external_source_db/`:**
- Same pattern as containerized, adapted for OpenShift target
- Empty `source_db` group
- Uses `target_ocp_bastion` for target operations

### 5. Documentation

**`README.md`:**
- Added "Source Environment (External Database)" prerequisites section
- Updated inventory structure section to include external source DB examples
- Added source DB configuration variables to key variables table
- Updated vault variables section with external source DB credential examples
- Added instructions for copying external source DB inventory examples

**`CLAUDE.md`:**
- Updated inventory groups section to explain external database configurations
- Added source DB variables to key variables list

## Configuration Pattern

### Managed Source Database (Default)
```yaml
source_db_type: managed  # default
# source_db inventory group must contain the database server
```

### External Source Database (RDS, Azure Database, etc.)
```yaml
source_db_type: external
source_pg_host: my-rds-instance.region.rds.amazonaws.com
source_pg_port: 5432
source_pg_ssl_mode: require
source_pg_admin_user: postgres
source_pg_admin_password: "{{ source_pg_admin_password }}"

# Per-component overrides in source.yml:
controller_pg_host: "{{ source_pg_host }}"
controller_pg_user: awx
controller_pg_password: "{{ controller_pg_password }}"
# ... same for hub and gateway
```

## Authentication Flow

1. **Managed DB**: Uses `become: postgres` and local socket authentication (existing behavior)
2. **External DB**: Uses network authentication with credentials passed via:
   - `PGPASSWORD` environment variable (avoids password in command line)
   - `PGSSLMODE` environment variable for SSL connection enforcement
   - `-h`, `-p`, `-U` flags for connection parameters

## Validation

Both playbooks pass syntax validation:
```bash
ansible-playbook --syntax-check migrate_rpm_to_containerized.yml
ansible-playbook --syntax-check migrate_rpm_to_openshift.yml
```

## Compatibility

- **Backward compatible**: Default `source_db_type: managed` preserves existing behavior
- **Consistent pattern**: Mirrors existing `target_db_type` implementation
- **Security**: Credentials passed via environment variables, never exposed in command line
- **SSL support**: Respects `source_pg_ssl_mode` for secure connections

## Testing Recommendations

1. Test with AWS RDS PostgreSQL 15
2. Test with Azure Database for PostgreSQL
3. Verify SSL connection enforcement with `source_pg_ssl_mode: require`
4. Confirm authentication with non-admin database users per component
5. Validate large database dumps work with external databases
6. Test timeout handling with `db_dump_timeout` for slow network connections
