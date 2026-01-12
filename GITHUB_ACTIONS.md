# GitHub Actions CI/CD Setup

This document explains how to configure GitHub Actions to automatically build and deploy your network configurations.

## Overview

Two workflows are included:

1. **Build Workflow** (`build.yml`) - Runs automatically on push/PR to validate configurations
2. **Deploy Workflow** (`deploy.yml`) - Manual trigger to deploy configurations to CloudVision

## Required GitHub Secrets

Navigate to your repository: **Settings → Secrets and variables → Actions → New repository secret**

### Required for Build Workflow (Optional - uses placeholders if not set)

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `ANSIBLE_USER` | Switch admin username | `admin` |
| `ANSIBLE_PASSWORD` | Switch admin password | Your switch password |
| `BGP_PASSWORD` | BGP peer group password (encrypted) | Generate with AVD |

### Required for Deploy Workflow (Mandatory)

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `ANSIBLE_USER` | Switch admin username | `admin` |
| `ANSIBLE_PASSWORD` | Switch admin password | Your switch password |
| `BGP_PASSWORD` | BGP peer group password (encrypted) | Generate with AVD |
| `CV_SERVER` | CloudVision server URL | `www.cv-prod-us-4.arista.io` |
| `CV_TOKEN` | CloudVision API token | Your JWT token |

## Setting Up GitHub Secrets

### 1. Add Ansible Credentials

```bash
# Navigate to: Settings → Secrets and variables → Actions
# Click: New repository secret

Name: ANSIBLE_USER
Secret: admin

Name: ANSIBLE_PASSWORD
Secret: <your-switch-password>
```

### 2. Generate BGP Password

Use AVD's encryption filter to generate a BGP password:

```python
python3 -c "from ansible_collections.arista.avd.plugins.filter.encrypt import FilterModule; print(FilterModule().generate_password('your_password', 'bgp'))"
```

Add the output as:
```
Name: BGP_PASSWORD
Secret: <generated-encrypted-password>
```

### 3. Add CloudVision Credentials

Get your CloudVision API token:
1. Log in to CloudVision portal
2. Navigate to Settings → Access Control → Service Accounts
3. Create/use service account and generate API token

Add to GitHub Secrets:
```
Name: CV_SERVER
Secret: www.cv-prod-us-4.arista.io

Name: CV_TOKEN
Secret: <your-cloudvision-jwt-token>
```

## GitHub Environments (Optional but Recommended)

For the deploy workflow, you can set up environments for additional protection:

1. Navigate to **Settings → Environments**
2. Create environments: `production` and `staging`
3. Configure protection rules:
   - **Required reviewers**: Add team members who must approve deployments
   - **Wait timer**: Add delay before deployment
   - **Deployment branches**: Restrict which branches can deploy

## Workflow Usage

### Build Workflow (Automatic)

Runs automatically when you:
- Push to `main` or `develop` branches
- Create a pull request to `main`

**What it does:**
- Installs Ansible and AVD collection
- Generates device configurations
- Uploads configurations as artifacts (available for 30 days)
- Shows configuration summary in PR

**View results:**
- Go to **Actions** tab in GitHub
- Click on the workflow run
- Download artifacts: `avd-configurations`

### Deploy Workflow (Manual)

Trigger manually when ready to deploy:

1. Go to **Actions** tab
2. Select **Deploy to CloudVision** workflow
3. Click **Run workflow**
4. Select:
   - **Environment**: `production` or `staging`
   - **Run Change Control**: Enable/disable automatic change control
5. Click **Run workflow**

**What it does:**
- Generates configurations
- Deploys to CloudVision
- Creates change control (if enabled)
- Uploads deployment artifacts (available for 90 days)

## Viewing Workflow Results

### Check Workflow Status

```bash
# Using GitHub CLI (if installed)
gh run list --workflow=build.yml
gh run list --workflow=deploy.yml

# View specific run
gh run view <run-id>
```

### Download Artifacts

From the Actions tab:
1. Click on a completed workflow run
2. Scroll to **Artifacts** section
3. Download `avd-configurations` or `deployment-artifacts`

Or use GitHub CLI:
```bash
gh run download <run-id>
```

## Troubleshooting

### Build Workflow Fails

**Common issues:**
- **Syntax errors in YAML**: Check group_vars files for proper YAML syntax
- **Missing variables**: Ensure all required variables are defined
- **AVD collection errors**: Check AVD requirements are met

**View detailed logs:**
```bash
gh run view <run-id> --log-failed
```

### Deploy Workflow Fails

**Common issues:**
- **Missing secrets**: Ensure all required secrets are configured
- **Invalid CV token**: Token may be expired or invalid
- **Network connectivity**: GitHub runners need internet access to CloudVision

**Check secrets:**
- Navigate to Settings → Secrets and variables → Actions
- Verify all required secrets are present

### Re-run Failed Workflow

1. Go to Actions tab
2. Click on failed workflow
3. Click **Re-run jobs** → **Re-run failed jobs**

## Security Best Practices

1. **Use GitHub Environments** for production deployments with required approvals
2. **Rotate secrets regularly**:
   - CloudVision tokens should be rotated quarterly
   - Update secrets in GitHub when changed
3. **Limit repository access**: Only grant write access to trusted team members
4. **Review deployment logs**: Check Actions logs after each deployment
5. **Use branch protection**: Require PR reviews before merging to main

## Advanced Configuration

### Customize Workflows

Edit workflow files in `.github/workflows/`:
- `build.yml` - Modify build triggers, Python version, or artifact retention
- `deploy.yml` - Add pre/post-deployment steps, notifications, etc.

### Add Slack Notifications

Add to workflow after deploy step:

```yaml
- name: Notify Slack
  if: always()
  uses: slackapi/slack-github-action@v1
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    payload: |
      {
        "text": "Deployment ${{ job.status }}: ${{ github.repository }}"
      }
```

### Schedule Regular Builds

Add to `build.yml` to run daily validation:

```yaml
on:
  schedule:
    - cron: '0 8 * * *'  # Run at 8 AM UTC daily
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
```

## Cost Considerations

- **Public repositories**: GitHub Actions is free with unlimited minutes
- **Private repositories**: 2,000 free minutes/month, then pay-as-you-go
- **Typical usage**:
  - Build workflow: ~3-5 minutes per run
  - Deploy workflow: ~5-10 minutes per run

Monitor usage: **Settings → Billing → Plans and usage**

## Support

- GitHub Actions docs: https://docs.github.com/en/actions
- Arista AVD docs: https://avd.arista.com
- GitHub Actions status: https://www.githubstatus.com
