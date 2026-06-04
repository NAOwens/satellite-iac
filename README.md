# satellite-iac

Infrastructure as Code for Red Hat Satellite. Provides a pull/deploy workflow to export a running Satellite instance's configuration to YAML and restore it to a new or replacement Satellite.

## Overview

This project follows the same pull/deploy IaC pattern used in `aap-iac` and `sno-iac`:

- **Pull** — connect to a running Satellite instance, export its configuration to YAML files via the Foreman and Katello REST APIs, and commit them to GitHub
- **Deploy** — read the exported YAML files and apply them to a target Satellite in the correct dependency order using the `redhat.satellite` collection

Exported files are stored under `iac-sat-exports/` and committed to this repository so they can be version-controlled and used to reproduce the Satellite configuration on demand.

## Playbooks

### `pull_satellite_resources.yml`

Connects to a running Satellite instance and exports its configuration to `iac-sat-exports/`. At the end of the run the exported files are committed and pushed to this GitHub repository. Only exports changed files — if nothing changed since the last run, no commit is made.

**What is exported:**

| Directory | Contents | API |
|---|---|---|
| `organizations/` | Organization definitions | Foreman `/api/v2/organizations` |
| `locations/` | Location definitions | Foreman `/api/v2/locations` |
| `domains/` | DNS domain definitions | Foreman `/api/v2/domains` |
| `lifecycle_environments/` | Lifecycle environments per organization | Katello `/katello/api/v2/environments` |
| `hostgroups/` | Host group definitions | Foreman `/api/v2/hostgroups` |
| `activation_keys/` | Activation keys per organization | Katello `/katello/api/v2/activation_keys` |
| `subscriptions/` | Subscription pools per organization | Katello `/katello/api/v2/organizations/{id}/subscriptions` |
| `redhat_repositories/` | Enabled Red Hat repositories per organization | Katello `/katello/api/v2/repositories` |
| `products/` | Products (Red Hat and custom) per organization | Katello `/katello/api/v2/products` |
| `sync_status/` | Repository sync status summary (name, product, last sync time) | Derived from repository data |
| `auth_sources/` | LDAP/AD authentication source configs (no bind passwords) | Foreman `/api/v2/auth_source_ldaps` |
| `users/` | User accounts | Foreman `/api/v2/users` |

**What is NOT exported:**

- Passwords (LDAP bind passwords, user passwords)
- Red Hat subscription manifest (must be uploaded separately)
- Content Views and Composite Content Views
- Sync plans and schedules
- Smart Proxies / Capsule configuration
- Host records

**Required variables (pass via AAP survey or Extra Vars):**

| Variable | Description |
|---|---|
| `survey_sat_host` | URL of the source Satellite (e.g. `https://sat.example.com`) |
| `sat_admin_user` | Satellite admin username |
| `sat_admin_pat` | Satellite admin password or Personal Access Token |
| `git_token_variable` | GitHub personal access token with write access to this repository |

**Example execution:**
```bash
ansible-playbook playbooks/pull_satellite_resources.yml \
  -e "survey_sat_host=https://sat.example.com" \
  -e "sat_admin_user=admin" \
  -e "sat_admin_pat=<token>" \
  -e "git_token_variable=<github-token>"
```

---

### `deploy_satellite_resources.yml`

Reads the exported YAML files from `iac-sat-exports/` and applies them to a target Satellite in the correct dependency order using the `redhat.satellite` Ansible collection.

**Deployment order:**

Resources are deployed in strict dependency order — each step's prerequisites must exist before it runs:

1. **Organizations** — required by every other resource; deployed first
2. **Locations** — required by Hostgroups and provisioning resources
3. **Domains** — DNS domain definitions; referenced by Hostgroups
4. **LDAP/AD Authentication Sources** — must exist before users that reference them
5. **Users** — `admin` is always skipped (already exists on any Satellite)
6. **Custom Products** — only non-Red Hat products (identified by absence of `cp_id`); Red Hat products are created automatically when repositories are enabled from the manifest
7. **Red Hat Repositories** — enabled using `content_label` (the stable CDN identifier) rather than display name, which avoids matching errors when product names contain version strings
8. **Lifecycle Environments** — `Library` is always skipped (present by default); all others deployed with their `prior` chain preserved
9. **Activation Keys** — reference Lifecycle Environments and Content Views
10. **Hostgroups** — reference Organizations and Locations

**Required variables:**

| Variable | Description |
|---|---|
| `survey_sat_host` | URL of the target Satellite (e.g. `https://sat-new.example.com`) |
| `sat_admin_user` | Satellite admin username |
| `sat_admin_pat` | Satellite admin password or Personal Access Token |

**Example execution:**
```bash
ansible-playbook playbooks/deploy_satellite_resources.yml \
  -e "survey_sat_host=https://sat-new.example.com" \
  -e "sat_admin_user=admin" \
  -e "sat_admin_pat=<token>"
```

**Post-deploy manual steps:**

After the playbook completes, the following must be done manually before the Satellite is fully operational:

1. **Upload the Red Hat subscription manifest** — required before repositories can sync. Download from `access.redhat.com` and upload via `Content → Subscriptions → Manage Manifest`.
2. **Update LDAP bind passwords** — auth source bind credentials are not exported. Edit each LDAP auth source and set the bind password.
3. **Sync repositories** — trigger a sync for all enabled repositories (or configure a sync plan).
4. **Publish and promote Content Views** — Content Views are not deployed by this playbook. Create, publish, and promote them after repositories have synced.
5. **Attach subscriptions to Activation Keys** — if not using Simple Content Access (SCA), attach the appropriate subscription pools to each key manually.

## Export Structure

```
iac-sat-exports/
├── activation_keys/
│   └── activation_keys.yml
├── auth_sources/
│   └── auth_sources.yml
├── domains/
│   └── domains.yml
├── hostgroups/
│   └── hostgroups.yml
├── lifecycle_environments/
│   └── lifecycle_environments.yml
├── locations/
│   └── locations.yml
├── organizations/
│   └── organizations.yml
├── products/
│   └── products.yml
├── redhat_repositories/
│   └── redhat_repositories.yml          # Full repo objects including content_label
├── subscriptions/
│   └── subscriptions.yml                # Reference only — manifest required to restore
├── sync_status/
│   └── sync_status.yml                  # Summary report: name, product, last sync time
└── users/
    └── users.yml
```

## Collections Required

```bash
ansible-galaxy collection install -r requirements.yml
```

`requirements.yml` installs `redhat.satellite`. This collection requires the `apypie` Python library on the execution node:

```bash
pip install apypie
```
