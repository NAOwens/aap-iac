# AAP Infrastructure as Code (AAP-IaC)

A comprehensive Ansible automation solution for managing Ansible Automation Platform (AAP) resources through Infrastructure as Code (IaC) principles. This repository provides playbooks to export AAP resources and deploy them to other environments for consistent, version-controlled resource management.

## Overview

The AAP-IaC project enables organizations to:
- **Export** AAP Controller configurations as YAML files
- **Version control** all AAP resource definitions
- **Deploy** resources consistently across multiple environments
- **Manage** organizations, users, teams, projects, credentials, inventories, job templates, workflows, and notification templates

## Repository Structure

```
aap-iac/
├── README.md                          # This file
├── requirements.yml                   # Ansible collection dependencies
└── playbooks/
    ├── pull_aap_resources.yml         # Export/backup playbook
    └── deploy_aap_resources.yml       # Deploy/restore playbook
```

## Prerequisites

- Ansible 2.9+ installed
- Python 3.6+
- Access to Ansible Automation Platform Controller
- Valid AAP Controller credentials
- Required Ansible collections (see below)

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/NAOwens/aap-iac.git
cd aap-iac
```

### 2. Install Required Collections

Install the required Ansible collections specified in `requirements.yml`:

```bash
ansible-galaxy collection install -r requirements.yml
```

### 3. Set Environment Variables

Set your AAP Controller credentials and URL:

```bash
export AAP_USERNAME="your_aap_username"
export AAP_PASSWORD="your_aap_password"
export AAP_CONTROLLER_URL="https://your-controller.example.com"
```

## Playbooks

### 1. `pull_aap_resources.yml` - Export/Backup Playbook

**Purpose:** Extracts all AAP Controller resources and exports them as YAML files for version control.

**What it exports:**
- Organizations
- Users
- Teams
- Credentials (metadata only - sensitive data excluded)
- Projects
- Inventories
- Job Templates
- Workflow Templates
- Notification Templates

**Output:** Creates timestamped YAML files in the `./iac-exports/` directory structure

**Key Features:**
- Validates AAP Controller credentials
- Creates organized directory structure
- Generates export summary report
- Excludes sensitive credential data for security
- Timestamped exports for historical tracking
- Error handling with graceful degradation

**Usage:**

```bash
ansible-playbook playbooks/pull_aap_resources.yml
```

**Optional variables:**

```bash
ansible-playbook playbooks/pull_aap_resources.yml \
  -e "aap_controller_url=https://custom-controller.com" \
  -e "validate_certs=false"
```

**Output Structure:**
```
iac-exports/
├── organizations/
│   └── organizations_YYYYMMDDTHHMMSS.yml
├── users/
│   └── users_YYYYMMDDTHHMMSS.yml
├── teams/
│   └── teams_YYYYMMDDTHHMMSS.yml
├── credentials/
│   └── credentials_metadata_YYYYMMDDTHHMMSS.yml
├── projects/
│   └── projects_YYYYMMDDTHHMMSS.yml
├── inventories/
│   └── inventories_YYYYMMDDTHHMMSS.yml
├── job_templates/
│   └── job_templates_YYYYMMDDTHHMMSS.yml
├── workflows/
│   └── workflows_YYYYMMDDTHHMMSS.yml
├── notification_templates/
│   └── notifications_YYYYMMDDTHHMMSS.yml
└── EXPORT_SUMMARY_YYYYMMDDTHHMMSS.md
```

---

### 2. `deploy_aap_resources.yml` - Deploy Playbook

**Purpose:** Deploys AAP resources from exported YAML files to an AAP Controller instance.

**What it deploys:**
- Organizations
- Users
- Teams
- Projects

**Input:** Reads from `./iac-exports/` directory (output of `pull_aap_resources.yml`)

**Key Features:**
- Validates required credentials
- Checks source directory exists
- Creates or updates resources in idempotent manner
- Error handling for graceful failure recovery
- Displays deployment summary upon completion
- Supports custom AAP Controller URLs

**Usage:**

```bash
ansible-playbook playbooks/deploy_aap_resources.yml
```

**Optional variables:**

```bash
ansible-playbook playbooks/deploy_aap_resources.yml \
  -e "aap_controller_url=https://target-controller.com" \
  -e "validate_certs=false" \
  -e "iac_source_dir=./iac-exports"
```

**Deployment Summary Output:**
```
Deployment Summary:
- Organizations created/updated: X
- Users created/updated: X
- Teams created/updated: X
- Projects created/updated: X
```

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `AAP_USERNAME` | AAP Controller username | Required |
| `AAP_PASSWORD` | AAP Controller password | Required |
| `AAP_CONTROLLER_URL` | AAP Controller URL | `https://controller.example.com` |
| `validate_certs` | Validate SSL certificates | `true` |

### Playbook Variables

#### pull_aap_resources.yml

| Variable | Description | Default |
|----------|-------------|---------|
| `aap_controller_host` | Controller hostname | From environment or default |
| `aap_controller_username` | Controller username | From `AAP_USERNAME` env var |
| `aap_controller_password` | Controller password | From `AAP_PASSWORD` env var |
| `aap_controller_validate_certs` | Validate SSL certs | `true` |
| `iac_output_dir` | Export output directory | `./iac-exports` |

#### deploy_aap_resources.yml

| Variable | Description | Default |
|----------|-------------|---------|
| `aap_controller_host` | Controller hostname | From environment or default |
| `aap_controller_username` | Controller username | From `AAP_USERNAME` env var |
| `aap_controller_password` | Controller password | From `AAP_PASSWORD` env var |
| `aap_controller_validate_certs` | Validate SSL certs | `true` |
| `iac_source_dir` | Input directory | `./iac-exports` |

## Common Workflows

### Backup AAP Configuration

```bash
# Set your AAP credentials
export AAP_USERNAME="admin"
export AAP_PASSWORD="password"
export AAP_CONTROLLER_URL="https://aap-prod.example.com"

# Export all resources
ansible-playbook playbooks/pull_aap_resources.yml

# Review and commit to git
git add iac-exports/
git commit -m "Backup AAP production configuration"
git push
```

### Migrate Resources Between Environments

```bash
# Export from production
export AAP_USERNAME="admin"
export AAP_PASSWORD="prod_password"
export AAP_CONTROLLER_URL="https://aap-prod.example.com"

ansible-playbook playbooks/pull_aap_resources.yml

# Deploy to staging
export AAP_USERNAME="admin"
export AAP_PASSWORD="staging_password"
export AAP_CONTROLLER_URL="https://aap-staging.example.com"

ansible-playbook playbooks/deploy_aap_resources.yml
```

### Disaster Recovery

```bash
# Restore from git history
git checkout <commit-hash>

# Deploy to new AAP instance
export AAP_USERNAME="admin"
export AAP_PASSWORD="new_password"
export AAP_CONTROLLER_URL="https://aap-new.example.com"

ansible-playbook playbooks/deploy_aap_resources.yml
```

## Security Considerations

⚠️ **Important Security Notes:**

1. **Credential Management:**
   - Sensitive credential data (passwords, API tokens) is **NOT** exported by the pull playbook
   - Credentials must be re-created manually or through secure automation
   - Consider using Ansible Vault for storing sensitive variables

2. **Environment Variables:**
   - Never commit AAP credentials to git
   - Use `.gitignore` to exclude credential files
   - Consider using credential management tools (AWS Secrets Manager, HashiCorp Vault, etc.)

3. **SSL Certificate Validation:**
   - Always validate SSL certificates in production (`validate_certs: true`)
   - Only disable certificate validation for development/testing

4. **Version Control:**
   - Use `.gitignore` to exclude:
     ```
     iac-exports/credentials/
     .env
     *.vault
     ```

## Troubleshooting

### Connection Issues

**Error:** `Failed to connect to AAP Controller`

**Solution:**
- Verify AAP Controller URL: `curl https://your-controller.example.com`
- Check firewall/network connectivity
- Verify SSL certificates if validation is enabled

### Authentication Failures

**Error:** `Authentication failed`

**Solution:**
- Verify `AAP_USERNAME` and `AAP_PASSWORD` are set correctly
- Confirm user has appropriate permissions in AAP
- Check AAP Controller logs for authentication errors

### Missing Collections

**Error:** `ERROR! Collection awx.awx not found`

**Solution:**
```bash
ansible-galaxy collection install -r requirements.yml --force
```

### Directory Not Found

**Error:** `Check IaC source directory exists - FAILED`

**Solution:**
- First run `pull_aap_resources.yml` to create export directory
- Verify path in `iac_source_dir` variable

## Ansible Collections

This project uses the following Ansible collections:

- **awx.awx** (≥24.0.0) - AWX collection for AAP resource management
- **ansible.controller** - Ansible Controller modules for resource deployment

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -m 'Add improvement'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

## Support

For issues and questions:

1. Check existing GitHub issues
2. Review AAP documentation: https://docs.ansible.com/automation-controller/
3. Consult Ansible collection documentation
4. Open a new GitHub issue with detailed information

## License

This project is licensed under the MIT License - see LICENSE file for details.

## Disclaimer

This project is provided as-is. Always test in non-production environments before deploying to production AAP instances. Backup your AAP Controller configuration before running deployment playbooks.

---

**Last Updated:** 2026-05-08

**Repository:** https://github.com/NAOwens/aap-iac
