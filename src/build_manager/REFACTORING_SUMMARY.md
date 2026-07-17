# Build Manager Credential & Directory Refactoring — Summary

## Overview

This refactoring separates **S3/MinIO credentials** and **MinIO/Registry deployment** from the
`prepare_oim` domain and moves them exclusively to the `build_manager` domain. It also introduces
dedicated input/output directory variables (`input_project_dir`, `output_project_dir`) so that
`build_manager` and all downstream playbooks use variable-driven paths instead of hardcoded ones.

---

## What Changed

### 1. Credential Ownership Transfer

| Credential          | Before (prepare_oim)             | After (build_manager)                      |
|---------------------|----------------------------------|--------------------------------------------|
| `s3_secret_key`     | In `omnia_config_credentials.yml`| In `build_manager_credentials.yml` only    |
| `s3_access_id`      | In `omnia_config_credentials.yml`| In `build_manager_credentials.yml` only    |
| `provision_password`| In `omnia_config_credentials.yml`| In `build_manager_credentials.yml` (mandatory) |
| `pulp_password`     | In `omnia_config_credentials.yml`| Stays in `omnia_config_credentials.yml`    |

**Key change:** `prepare_oim` no longer prompts for S3 credentials. Running `omnia.sh --tags prepare_oim`
will only ask for `pulp_password` and optional Docker credentials.

#### Files changed:
- `credential_utility/roles/update_config/vars/main.yml` — removed `s3_secret_key` and `s3_access_id`
  from `prepare_oim` tag; added `provision_password` as mandatory in `build_manager` tag.
- `credential_utility/roles/create_config/templates/omnia_credential.j2` — removed S3 fields.
- `credential_utility/roles/create_config/templates/build_manager_credential.j2` — added
  `provision_password`.
- `credential_utility/roles/create_config/vars/main.yml` — Build Manager credential file entry
  at `{{ input_project_dir }}/build_manager_credentials.yml`.

### 2. MinIO/Registry Deployment Moved to build_manager

The following files were **git rm**'d from `prepare_oim` (they now live in `build_manager`):

```
DELETED  prepare_oim/roles/deploy_containers/openchami/tasks/configs/minio.yml
DELETED  prepare_oim/roles/deploy_containers/openchami/tasks/configs/registry.yml
DELETED  prepare_oim/roles/deploy_containers/openchami/tasks/configs/s3_bucket.yml
DELETED  prepare_oim/roles/deploy_containers/openchami/tasks/configs/policy_update.yml
DELETED  prepare_oim/roles/deploy_containers/openchami/templates/minio/minio.service.j2
DELETED  prepare_oim/roles/deploy_containers/openchami/templates/registry/registry.service.j2
DELETED  prepare_oim/roles/deploy_containers/openchami/templates/s3/s3cfg.j2
DELETED  prepare_oim/roles/deploy_containers/openchami/templates/s3/s3-public-read-boot.json.j2
DELETED  prepare_oim/roles/deploy_containers/openchami/templates/s3/s3-public-read-efi.json.j2
```

The build_manager domain now handles:
- MinIO container deployment (`build_manager/roles/deploy_minio/`)
- Registry container deployment
- S3 bucket creation and ACL policies

#### Files changed:
- `prepare_oim/roles/deploy_containers/openchami/tasks/configs/main.yml` — removed imports of
  `minio.yml`, `registry.yml`, `s3_bucket.yml`, `policy_update.yml`.
- `prepare_oim/roles/deploy_containers/openchami/tasks/deploy_openchami.yml` — removed
  `s3_access_id` / `s3_secret_key` fact-setting block (lines 156-160).

### 3. Input/Output Directory Variables

#### New variables:
- `omnia_output_dir` — `/opt/omnia/output` (in `include_input_dir/vars/main.yml`)
- `output_project_dir` — `{{ omnia_output_dir }}/{{ project_name }}` (set as fact in
  `include_input_dir/tasks/main.yml`, parallel to `input_project_dir`)

#### How it works:
1. `omnia.sh` creates the output directory: `mkdir -p /opt/omnia/output/project_default`
2. `omnia.sh` copies build_manager input files to `{{ input_project_dir }}/build_manager/`
   (no credential files — those are loaded dynamically by the credential utility)
3. `include_input_dir` role sets both `input_project_dir` and `output_project_dir` as facts
4. `build_manager.yml` uses `output_project_dir` for all output paths

#### Files changed:
- `playbooks/utils/roles/include_input_dir/vars/main.yml` — added `omnia_output_dir`.
- `playbooks/utils/roles/include_input_dir/tasks/main.yml` — added `output_project_dir` fact.
- `main/omnia.sh` (`post_setup_config`) — creates output dir and copies build_manager inputs.

### 4. build_manager.yml Playbook Updates

- **Step 3** (repo_manager pre-check): `_repo_status_path` default now uses `output_project_dir`.
- **Step 8** (write build_status): Uses `_output_base` / `_output_domain` from `output_project_dir`.
- **write_build_status.yml** role: Writes to `{{ output_project_dir }}/build_manager/build_status.yml`.
- Removed EFI fields and metadata (omnia_version, timestamp) from `build_status_data` output.
- `build_manager_config.yml`: `repo_manager_output` default updated to use `output_project_dir`.

### 5. Upgrade & Rollback Compatibility

#### Upgrade (`restore_omnia_config_credentials.yml`):
- Removed S3 creds from main `set_fact` (no longer in omnia_config_credentials.yml).
- **Migration block preserved**: If upgrading from an older version that stored S3 creds in
  `omnia_config_credentials.yml`, they are automatically migrated to `build_manager_credentials.yml`.

#### Rollback (`load_rollback_credentials.yml`):
- Removed S3 creds from main `set_fact`.
- Loads S3 + `provision_password` from `build_manager_credentials.yml` when available.
- Fallback: reads S3 from old `omnia_config_credentials.yml` for backward compat with older backups.

#### Files changed:
- `upgrade/roles/import_input_parameters/templates/omnia_config_credentials.yml.j2` — removed S3.
- `upgrade/roles/import_input_parameters/templates/build_manager_credentials.yml.j2` — added
  `provision_password`.
- `upgrade/roles/import_input_parameters/tasks/restore_omnia_config_credentials.yml` — removed S3
  from main set_fact; migration block handles old→new.
- `rollback/playbooks/load_rollback_credentials.yml` — removed S3 from main set_fact; loads from
  build_manager_credentials.yml.

---

## How It Works (End-to-End Flow)

```
┌─────────────────────────────────────────────────────────┐
│                     omnia.sh                            │
│  1. Creates /opt/omnia/output/project_default/          │
│  2. Copies build_manager input → input/project_default/ │
│     build_manager/ (no credential files)                │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              credential_utility                         │
│  Tag: build_manager                                     │
│  Prompts: s3_secret_key, s3_access_id*, provision_pass  │
│  Writes: build_manager_credentials.yml (vault-encrypted)│
│  (* conditional: only if provider == 'powerscale')      │
│                                                         │
│  Tag: prepare_oim                                       │
│  Prompts: pulp_password, docker_username/password*      │
│  Writes: omnia_config_credentials.yml (NO S3 creds)     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              include_input_dir role                      │
│  Sets facts:                                            │
│    input_project_dir  = /opt/omnia/input/project_default│
│    output_project_dir = /opt/omnia/output/project_default│
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              build_manager.yml                          │
│  1. Loads build_manager_credentials.yml (S3 + provision)│
│  2. Reads repo_status.yml from output_project_dir       │
│  3. Deploys MinIO + Registry (via deploy_minio role)    │
│  4. Builds images (x86_64 local, aarch64 via SSH)       │
│  5. Writes build_status.yml → output_project_dir/       │
│     build_manager/                                      │
└─────────────────────────────────────────────────────────┘
```

### Credential File Locations

```
/opt/omnia/input/project_default/
├── omnia_config_credentials.yml          ← pulp, bmc, docker, slurm, etc.
├── .omnia_config_credentials_key
├── build_manager_credentials.yml         ← s3_access_id, s3_secret_key, provision_password
├── .build_manager_credentials_key
├── build_manager/                        ← build_manager input configs (NOT creds)
│   ├── build_manager_config.yml
│   └── storage_config.yml
└── ...other input files...

/opt/omnia/output/project_default/
├── repo_manager/
│   └── repo_status.yml                   ← consumed by build_manager Step 3
└── build_manager/
    └── build_status.yml                  ← written by build_manager Step 8
```

---

## Backward Compatibility

- **No breaking changes** for users who don't use build_manager.
- **prepare_oim** no longer prompts for S3 creds — one fewer password to enter.
- **Upgrade from older versions**: S3 creds in old `omnia_config_credentials.yml` are automatically
  migrated to `build_manager_credentials.yml`.
- **Rollback**: Falls back to reading S3 from old credential file if
  `build_manager_credentials.yml` doesn't exist yet.
