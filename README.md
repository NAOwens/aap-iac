# Ansible Automation Platform (AAP) Infrastructure as Code (IaC)

This repository contains the Infrastructure as Code (IaC) playbooks required to backup, migrate, and deploy Ansible Automation Platform (AAP) resources. It is specifically designed to handle migrations from older, standalone AAP architectures into a modernized **AAP 2.6 OpenShift environment** utilizing the unified Automation Gateway.

## Overview

This toolset is broken down into four primary playbooks: two for pulling (exporting) data from your existing environment, and two for deploying (restoring/migrating) that data into your new environment. 

By leveraging native Ansible modules and targeted API orchestration, these playbooks dynamically map relational database IDs to string names, ensuring your exported configurations are 100% portable across different AAP instances.

---

## 1. Controller Pull Playbook (`pull_aap_resources.yml`)

**Purpose:** Extracts all primary Controller components and commits them to version control.
**Exported Resources:** Organizations, Users, Teams, Projects, Inventories, Inventory Sources, Hosts, Job Templates, Workflow Templates, and Notification Templates.
**Output Directory:** `./iac-exports/`

**Key Features:**
* Automatically translates rigid database IDs (e.g., `organization: 2`) into portable string names (e.g., `organization: "Org1"`).
* Safely excludes sensitive credential payloads (passwords, keys, tokens).
* Automatically commits and pushes the generated YAML files back to this GitHub repository.

**Required Variables / Environment Variables:**
* `CONTROLLER_HOST`
* `CONTROLLER_OAUTH_TOKEN`
* `github_token` (for committing exports)

---

## 2. Controller Deploy Playbook (`deploy_aap_resources.yml`)

**Purpose:** Deploys the Controller IaC YAML files into the new AAP 2.6 environment.

**Key Features:**
* Uses `ansible.platform` and `ansible.controller` collections.
* Deploys the foundational RBAC structure (Organizations, Users, Teams) before attaching execution resources.
* Includes intelligent omission logic (using `default(omit)`) to bypass strict API validation errors on empty variables or null relationships.

**Required Extra Vars:**
* `new_aap_gateway_url` (The AAP 2.6 Gateway Route)
* `my_aap_token` (OAuth2 token generated from the Gateway)
* `aap_password` (Required for the `ansible.platform` modules executing Basic Auth)

---

## 3. Hub Pull Playbook (`pull_aap_hub_resources.yml`)

**Purpose:** Extracts structural configurations from your standalone Private Automation Hub.
**Exported Resources:** Namespaces, Execution Environment Repositories, and Remote Collection Registries.
**Output Directory:** `./iac-hub-exports/`

**Key Features:**
* Uses `ansible.builtin.uri` to interface directly with the Galaxy NG / Pulp backend.
* Authenticates using the strict `Token <hash>` header required by standalone Hubs.

**Required Variables / Environment Variables:**
* `ah_host` (e.g., `https://hub.example.com`)
* `ah_token` (Hub-specific Token)

---

## 4. Hub Deploy Playbook (`deploy_aap_hub_resources.yml`)

**Purpose:** Deploys the Hub IaC YAML files into the new AAP 2.6 environment via the Automation Gateway.

**Key Features:**
* Specially crafted to orchestrate the backend **Pulp API** directly through the AAP Gateway proxy.
* Bypasses the Gateway's strict `POST` method blocks and `ansible.hub` header limitations by forcing the `Authorization: Bearer` header on native `uri` tasks.
* Highly idempotent: Automatically catches and safely ignores `409 Already Exists` and `400 Bad Request` uniqueness conflicts.

**Required Extra Vars:**
* `new_aap_gateway_url` (The AAP 2.6 Gateway Route)
* `my_aap_token` (OAuth2 token generated from the Gateway)

---

## Workflow / Order of Operations

When performing a full environment sync or migration, execute the playbooks in the following order:

1. **Pull Hub Data** (`pull_aap_hub_resources.yml`) - Extracts Hub architecture to Git.
2. **Pull Controller Data** (`pull_aap_resources.yml`) - Extracts Controller architecture to Git.
3. *Manual Step:* Create missing Credentials in the new environment (e.g., Satellite Credentials, GitHub PATs).
4. **Deploy Hub Data** (`deploy_aap_hub_resources.yml`) - Builds the Hub namespaces and repos.
5. *Manual Step:* Push your custom Execution Environment container images (via Podman/Docker) to the newly created Hub repositories.
6. **Deploy Controller Data** (`deploy_aap_resources.yml`) - Builds the Controller configuration and links Job Templates to the new Hub Execution Environments.

## Known Architectural Quirks Managed by this Codebase
* **Gateway Token Routing:** AAP 2.6 collapses services behind a single Gateway proxy. These playbooks handle the required transition from standard Hub `Token` headers to Gateway `Bearer` headers.
* **Empty Variables:** Replaces empty strings or `---` YAML markers with `omit` to prevent 400 Bad Request errors.
* **Inventory Linking:** Prevents ambiguous duplicate inventory lookup failures by defaulting missing inventories to force a prompt-on-launch. 

## Manual steps needed on the new AAP server after running deploy
1. Had to manually add my host filter and Source variables to Satellite Inventory source in order to get it to sync the inventory.