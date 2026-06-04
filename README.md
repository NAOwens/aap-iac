# Ansible Automation Platform (AAP) Infrastructure as Code

This repository contains IaC playbooks for exporting, migrating, and restoring Ansible Automation Platform configurations. Designed for migrations into **AAP 2.6+ on OpenShift** with the unified Automation Gateway, but works equally well for same-version backup/restore.

## Overview

Four playbooks form two pull/deploy pairs — one for the Controller, one for Hub. Run the pull playbooks on a source environment to export configuration to Git, then run the deploy playbooks against a target environment to restore it.

```
Pull  ──► Git ──► Deploy
```

## Playbooks

### `pull_aap_resources.yml` — Controller Pull

Connects to a running AAP Controller and exports its configuration to `iac-exports/`, then commits and pushes to this repository. Only commits when files actually changed — no-op runs produce no git noise.

**What is exported:**

| Directory | Contents |
|---|---|
| `organizations/` | Organization definitions |
| `users/` | User accounts |
| `teams/` | Team definitions |
| `credentials/` | Credential metadata — names, types, org assignments (no secrets) |
| `projects/` | SCM project definitions |
| `inventories/inventories.yml` | Inventory definitions |
| `inventories/inventory_sources.yml` | Dynamic inventory source configs |
| `inventories/hosts.yml` | Static host entries |
| `execution_environments/` | EE name, image URL, and pull policy |
| `job_templates/` | Job template definitions including all relationships |
| `workflows/` | Workflow job template definitions |
| `notification_templates/` | Notification template configs |

**What is NOT exported:**
- Credential secrets (passwords, tokens, SSH keys)
- Workflow node structure and survey specs
- RBAC role assignments beyond org/team membership

**How relationships are stored:** The raw Controller API response — including `summary_fields` — is written directly to YAML. Human-readable names (org name, project name, inventory name, etc.) are read from `summary_fields` in the deploy playbook. No ID-to-name conversion is performed during pull.

**Required variables:**

| Variable | Source | Description |
|---|---|---|
| `CONTROLLER_HOST` | Environment variable | Controller API URL |
| `CONTROLLER_OAUTH_TOKEN` | Environment variable | OAuth2 token with admin access |
| `github_token` | Extra Var | GitHub PAT with write access to this repo |

**Example execution:**
```bash
ansible-playbook playbooks/pull_aap_resources.yml \
  -e "github_token=<token>"
# CONTROLLER_HOST and CONTROLLER_OAUTH_TOKEN set in environment
```

---

### `deploy_aap_resources.yml` — Controller Deploy

Clones this repository and deploys all exported Controller resources to a target AAP instance in strict dependency order. Failures propagate — a failed task stops the playbook rather than being silently ignored. The post-run summary reports `successful/total` counts for every resource type.

**Deployment order:**

1. **Organizations** (`ansible.platform`) — all other resources depend on these
2. **Users** (`ansible.platform`)
3. **Teams** (`ansible.platform`) — references organizations
4. **Credentials** (`ansible.controller`) — placeholder shells created with correct name, type, and org; secrets must be entered manually after deploy
5. **Projects** (`ansible.controller`) — references organizations
6. **Inventories** (`ansible.controller`) — references organizations
7. **Inventory Sources** — references inventories and credentials
8. **Hosts** — references inventories
9. **Execution Environments** (`ansible.controller`) — reads from exported `execution_environments/execution_environments.yml`; EE images must already exist in Hub
10. **Job Templates** (`ansible.controller`) — references projects, inventories, credentials, and EEs; sets `ask_inventory_on_launch: true` automatically when no default inventory is assigned
11. **Workflow Templates** (`ansible.controller`) — references organizations
12. **Notification Templates** (`ansible.controller`)

**Required variables:**

| Variable | Description |
|---|---|
| `new_aap_gateway_url` | AAP 2.6 Gateway URL |
| `my_aap_token` | OAuth2 token from the Gateway |

**Example execution:**
```bash
ansible-playbook playbooks/deploy_aap_resources.yml \
  -e "new_aap_gateway_url=https://gateway.example.com" \
  -e "my_aap_token=<token>"
```

**Post-deploy manual steps:**
1. **Enter credential secrets** — shells exist with the right type and org, but all secret fields are blank
2. **Reconfigure workflow nodes** — workflow structure, node linkage, and survey specs are not exported and must be rebuilt
3. **Verify inventory sources** — host filters and source variables may need adjustment after deploy

---

### `pull_aap_hub_resources.yml` — Hub Pull

Connects to a Private Automation Hub and exports its structural configuration to `iac-hub-exports/`, then commits and pushes to this repository. Only commits when files actually changed.

**What is exported:**

| Directory | Contents | API endpoint |
|---|---|---|
| `namespaces/` | Galaxy namespace definitions | `/api/galaxy/v3/namespaces/` |
| `execution_environments/` | EE repository definitions | `/api/galaxy/v3/plugin/execution-environments/repositories/` |
| `remotes/` | Collection remote registry configs | `/api/galaxy/_ui/v1/remotes/` |

**What is NOT exported:**
- Container image content (pushed separately via `podman push`)
- Collection tarballs and content

**Required variables:**

| Variable | Source | Description |
|---|---|---|
| `survey_ah_host` | Extra Var / Survey | Hub URL (e.g. `https://hub.example.com`) |
| `AH_TOKEN` | Extra Var | Hub API token |
| `github_token` | Extra Var | GitHub PAT with write access to this repo |

**Example execution:**
```bash
ansible-playbook playbooks/pull_aap_hub_resources.yml \
  -e "survey_ah_host=https://hub.example.com" \
  -e "AH_TOKEN=<token>" \
  -e "github_token=<github-token>"
```

---

### `deploy_aap_hub_resources.yml` — Hub Deploy

Clones this repository and deploys Hub resources to a target AAP 2.6 instance via the Automation Gateway proxy. Uses the Pulp API directly for EE repositories because the Gateway does not pass `POST` requests through to the Galaxy NG backend.

**Deployment order:**

1. **Namespaces** — Galaxy namespaces via `/api/galaxy/v3/namespaces/`
2. **EE Repositories** — Pulp container repositories created via `/api/galaxy/pulp/api/v3/repositories/container/container/`
3. **EE HREF map** — After creation, each repository is looked up by name and its `pulp_href` stored in a name-keyed dict (avoids fragile loop-index alignment if any item is skipped or fails)
4. **EE Distributions** — Pulp distributions that expose each repository through the Gateway, using the HREF map for reliable repository linkage
5. **Collection Remotes** — Remote collection registry configurations

**Required variables:**

| Variable | Description |
|---|---|
| `new_aap_gateway_url` | AAP 2.6 Gateway URL |
| `my_aap_token` | OAuth2 token from the Gateway |

**Example execution:**
```bash
ansible-playbook playbooks/deploy_aap_hub_resources.yml \
  -e "new_aap_gateway_url=https://gateway.example.com" \
  -e "my_aap_token=<token>"
```

**Post-deploy manual step:** Push EE container images to the newly created repositories:
```bash
podman push <image> gateway.example.com/<ee-name>:latest
```

---

## Full Migration Order

```
1. pull_aap_hub_resources.yml       Export Hub config to Git
2. pull_aap_resources.yml           Export Controller config to Git
3. deploy_aap_hub_resources.yml     Restore Hub namespaces, EE repos, remotes
4. (manual) podman push             Push EE images to new Hub
5. deploy_aap_resources.yml         Restore Controller config
6. (manual)                         Enter credential secrets
7. (manual)                         Rebuild workflow nodes and survey specs
```

---

## Export Structure

```
iac-exports/                              # Controller exports
├── EXPORT_SUMMARY.md
├── credentials/
│   └── credentials_metadata.yml          # Names/types/orgs — no secrets
├── execution_environments/
│   └── execution_environments.yml        # Controller-side EE references
├── inventories/
│   ├── hosts.yml
│   ├── inventories.yml
│   └── inventory_sources.yml
├── job_templates/
│   └── job_templates.yml
├── notification_templates/
│   └── notifications.yml
├── organizations/
│   └── organizations.yml
├── projects/
│   └── projects.yml
├── teams/
│   └── teams.yml
├── users/
│   └── users.yml
└── workflows/
    └── workflows.yml

iac-hub-exports/                          # Hub exports
├── EXPORT_SUMMARY.md
├── execution_environments/
│   └── execution_environments.yml        # Hub-side EE repository definitions
├── namespaces/
│   └── namespaces.yml
└── remotes/
    └── remotes.yml
```

---

## Architectural Notes

**Gateway token routing:** AAP 2.6 routes all services through a single Gateway proxy. Controller resources use `ansible.controller` with `controller_oauthtoken`. Hub Pulp API calls require `Authorization: Bearer <token>` explicitly — the Gateway strips standard Hub `Token` headers on proxied requests.

**Execution Environment split:** EEs exist in two places — as container image repositories in Hub (created by `deploy_aap_hub_resources.yml`) and as Controller-side pointers to those images (created by `deploy_aap_resources.yml`). Hub deploy must run before Controller deploy so images are reachable when job templates reference them.

**Relationship portability:** All API responses are exported with `summary_fields` intact, which contains human-readable names for every related object. The deploy playbook reads directly from `summary_fields` (e.g. `item.summary_fields.organization.name`) rather than numeric IDs, making exports portable across any AAP instance regardless of internal database state.

**Idempotency — 409 handling:** Hub deploy tasks accept HTTP 409 Conflict as non-fatal so re-running is safe. HTTP 400 Bad Request is not accepted — it indicates a genuinely malformed payload and will fail the task loudly.

**Credentials:** Credential shells are created with the correct name, type, and organization so job templates can reference them immediately. Secret fields are left blank and must be populated manually — secrets are never exported from the source environment.

**Known limitations:**
- Inventory source host filters and source variables may need manual adjustment after deploy (source-specific config not fully captured in export)
- Workflow node structure, approval nodes, and survey specs must be rebuilt manually — the `workflow_job_template` module creates the template shell only
