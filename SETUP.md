# Setup Instructions

This document explains how to configure the required secrets and credentials for this AVD dual-DC project.

## Deployment Options

You have two options for running the playbooks:

1. **GitHub Actions (Recommended)** - Automated CI/CD with GitHub runners
2. **Local Execution** - Run playbooks from your workstation

### Option 1: GitHub Actions (Recommended)

GitHub Actions provides automated builds and deployments without needing local setup.

**See [GITHUB_ACTIONS.md](GITHUB_ACTIONS.md) for complete setup instructions.**

**Quick start:**
- Configure GitHub Secrets with your credentials
- Push to main branch → automatic build
- Manual deploy trigger when ready

**Benefits:**
- No local Ansible installation required
- Automatic validation on every push/PR
- Audit trail of all deployments
- Team collaboration with approval workflows

### Option 2: Local Execution

For local development and testing, follow the instructions below.

## Initial Setup

### 1. Create Configuration Files from Templates

Copy the example files and customize them with your credentials:

```bash
# Copy deploy.yml template
cp deploy.yml.example deploy.yml

# Copy DC-L3LS-FABRIC.yml template
cp group_vars/DC-L3LS-FABRIC.yml.example group_vars/DC-L3LS-FABRIC.yml
```

### 2. Configure deploy.yml

Edit `deploy.yml` and replace the following placeholders:

- `YOUR_CLOUDVISION_SERVER` - Your CloudVision server URL (e.g., `www.cv-prod-us-4.arista.io`)
- `YOUR_CLOUDVISION_API_TOKEN_HERE` - Your CloudVision API token

To obtain a CloudVision API token:
1. Log in to your CloudVision portal
2. Navigate to Settings → Access Control → Service Accounts
3. Create a new service account or use an existing one
4. Generate an API token

**Note:** If you don't use CloudVision, you can uncomment the direct eAPI deployment section and comment out the CVaaS section.

### 3. Configure group_vars/DC-L3LS-FABRIC.yml

Edit `group_vars/DC-L3LS-FABRIC.yml` and replace:

- `YOUR_SWITCH_PASSWORD_HERE` - The password for the admin user on your switches
- `YOUR_BGP_PASSWORD_HERE` - BGP peer group passwords (used for overlay and underlay)

#### Generating BGP Passwords

BGP passwords should be encrypted using AVD's encryption filter. You can generate them with:

```python
python3 -c "from ansible_collections.arista.avd.plugins.filter.encrypt import FilterModule; print(FilterModule().generate_password('your_plaintext_password', 'bgp'))"
```

Or for testing/lab environments, you can use the same plaintext password for all three BGP peer groups.

### 4. Verify Configuration

Before running the playbooks, verify your inventory is correct:

```bash
ansible-inventory --graph
```

## Security Notes

**IMPORTANT:** The files `deploy.yml` and `group_vars/DC-L3LS-FABRIC.yml` are in `.gitignore` to prevent accidentally committing secrets to version control.

- Never commit these files with real credentials
- Use the `.example` templates for sharing configurations
- Consider using Ansible Vault for additional security in production environments

## Alternative: Using Ansible Vault (Production)

For production environments, consider encrypting sensitive variables with Ansible Vault:

```bash
# Encrypt the entire file
ansible-vault encrypt group_vars/DC-L3LS-FABRIC.yml

# Or encrypt specific variables
ansible-vault encrypt_string 'your_password' --name 'ansible_password'
```

Then run playbooks with:
```bash
ansible-playbook build.yml --ask-vault-pass
```

## Running the Playbooks

Once configured, you can:

```bash
# Build configurations only (no deployment)
ansible-playbook build.yml

# Build and deploy to CloudVision
ansible-playbook deploy.yml
```
