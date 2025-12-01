# Azure VM Control

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Azure%20VM%20Control-blue.svg)](https://github.com/marketplace/actions/azure-vm-control)
[![Integration](https://github.com/farooq-teqniqly/azure-vm-control/actions/workflows/integration.yml/badge.svg)](https://github.com/farooq-teqniqly/azure-vm-control/actions/workflows/integration.yml)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=farooq-teqniqly_azure-vm-control&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=farooq-teqniqly_azure-vm-control)



A GitHub Action that automates starting and stopping Azure Virtual Machines, designed specifically for managing self-hosted runners in CI/CD pipelines.

## Table of Contents

- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Getting Started](#getting-started)
- [Examples](#examples)
- [API Reference](#api-reference)
- [Benefits](#benefits)
- [License](#license)

## The Problem

Managing Azure Virtual Machines (VMs) used as GitHub self-hosted runners can be challenging:

- **Manual Operations**: Starting and stopping VMs requires manual intervention or separate automation
- **Cost Management**: VMs left running unnecessarily increase Azure costs
- **CI/CD Integration**: No seamless way to control VM lifecycle within GitHub workflows
- **Error-Prone**: Manual Azure CLI commands are prone to typos and inconsistent execution

## The Solution

Azure VM Control provides a simple, reliable way to start and deallocate Azure VMs directly from your GitHub workflows. It handles authentication, validation, and state checking automatically.

**Using OIDC Authentication (Recommended):**

```yaml
- name: Start VM Runner
  uses: farooq-teqniqly/azure-vm-control@v1
  with:
    azure_resource_group_name: 'my-resource-group'
    azure_vm_name: 'my-runner-vm'
    client_id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant_id: ${{ secrets.AZURE_TENANT_ID }}
    subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    operation: 'start'
```

**Using Credential-based Authentication:**

```yaml
- name: Start VM Runner
  uses: farooq-teqniqly/azure-vm-control@v1
  with:
    azure_resource_group_name: 'my-resource-group'
    azure_vm_name: 'my-runner-vm'
    azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
    operation: 'start'
```

## Getting Started

### Prerequisites

- An Azure subscription with a Virtual Machine
- Azure service principal with VM management permissions
- For OIDC auth: Federated credentials configured for your GitHub repository
- For credential-based auth: Service principal credentials JSON
- GitHub repository with secrets configured

#### Azure Authentication Setup

This action supports two authentication methods:

1. **OIDC Authentication (Recommended)** - Uses OpenID Connect for passwordless authentication
2. **Credential-based Authentication** - Uses Azure service principal credentials JSON

##### OIDC Authentication (Recommended)

OIDC authentication is more secure as it doesn't require storing long-lived credentials.

**Step 1: Create a service principal and configure federated credentials**

```bash
# Create a service principal (without credentials)
az ad sp create-for-rbac --name SERVICE_PRINCIPAL_NAME --role "Virtual Machine Contributor" --scopes /subscriptions/SUBSCRIPTION_ID/resourceGroups/RESOURCE_GROUP_NAME --create-cert=false
```

**Step 2: Configure federated credentials for GitHub Actions**

In the Azure Portal:
1. Navigate to Azure Active Directory > App registrations
2. Find your service principal and select it
3. Go to "Certificates & secrets" > "Federated credentials"
4. Click "Add credential"
5. Select "GitHub Actions deploying Azure resources"
6. Configure:
   - Organization: Your GitHub organization or username
   - Repository: Your repository name
   - Entity type: Branch, Pull request, Environment, or Tag
   - GitHub branch name: main (or your branch)
   - Name: A descriptive name for the credential

**Step 3: Store the following as GitHub secrets:**
- `AZURE_CLIENT_ID` - Application (client) ID
- `AZURE_TENANT_ID` - Directory (tenant) ID
- `AZURE_SUBSCRIPTION_ID` - Your Azure subscription ID

**Step 4: Add required permissions to your workflow:**

```yaml
permissions:
  id-token: write  # Required for OIDC
  contents: read
```

##### Credential-based Authentication

Create a service principal with the necessary permissions for VM management.

**Using PowerShell:**

```powershell
# Get the resource group scope
$scope = az group show -n RESOURCE_GROUP_NAME --query id -o tsv

# Create service principal with Virtual Machine Contributor role
az ad sp create-for-rbac --name SERVICE_PRINCIPAL_NAME --role "Virtual Machine Contributor" --scopes $scope --sdk-auth
```

**Using Azure CLI (Bash):**

```bash
# Get the resource group scope
SCOPE=$(az group show -n RESOURCE_GROUP_NAME --query id -o tsv)

# Create service principal with Virtual Machine Contributor role
az ad sp create-for-rbac --name SERVICE_PRINCIPAL_NAME --role "Virtual Machine Contributor" --scopes $SCOPE --sdk-auth
```

The `--sdk-auth` flag outputs the credentials in the JSON format required by the `azure_credentials` input. Copy this JSON output and store it as a GitHub secret named `AZURE_CREDENTIALS`.

### Installation

Add this action to your GitHub workflow YAML file:

**Using OIDC Authentication (Recommended):**

```yaml
name: Control Azure VM
on: [workflow_dispatch]

permissions:
  id-token: write  # Required for OIDC
  contents: read

jobs:
  control-vm:
    runs-on: ubuntu-latest
    steps:
      - name: Control VM
        uses: farooq-teqniqly/azure-vm-control@v1
        with:
          azure_resource_group_name: 'your-resource-group'
          azure_vm_name: 'your-vm-name'
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**Using Credential-based Authentication:**

```yaml
name: Control Azure VM
on: [workflow_dispatch]

jobs:
  control-vm:
    runs-on: ubuntu-latest
    steps:
      - name: Control VM
        uses: farooq-teqniqly/azure-vm-control@v1
        with:
          azure_resource_group_name: 'your-resource-group'
          azure_vm_name: 'your-vm-name'
          azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
```

### Basic Usage

#### Starting a VM

```yaml
- name: Start Azure VM
  uses: farooq-teqniqly/azure-vm-control@v1
  with:
    azure_resource_group_name: 'my-runners-rg'
    azure_vm_name: 'github-runner-01'
    azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
    operation: 'start'  # This is the default
```

#### Stopping a VM

```yaml
- name: Stop Azure VM
  uses: farooq-teqniqly/azure-vm-control@v1
  with:
    azure_resource_group_name: 'my-runners-rg'
    azure_vm_name: 'github-runner-01'
    azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
    operation: 'deallocate'
```

### Advanced Usage

#### Conditional VM Control

Control VMs based on workflow conditions:

```yaml
- name: Start VM for deployment
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  uses: farooq-teqniqly/azure-vm-control@v1
  with:
    azure_resource_group_name: 'prod-runners'
    azure_vm_name: 'deployment-runner'
    azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
    operation: 'start'

- name: Stop VM after deployment
  if: always()
  uses: farooq-teqniqly/azure-vm-control@v1
  with:
    azure_resource_group_name: 'prod-runners'
    azure_vm_name: 'deployment-runner'
    azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
    operation: 'deallocate'
```

#### Multiple VM Management

Manage multiple VMs in a single workflow:

```yaml
- name: Start multiple VMs
  uses: farooq-teqniqly/azure-vm-control@v1
  with:
    azure_resource_group_name: 'runners-rg'
    azure_vm_name: 'runner-01'
    azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}

- name: Start second VM
  uses: farooq-teqniqly/azure-vm-control@v1
  with:
    azure_resource_group_name: 'runners-rg'
    azure_vm_name: 'runner-02'
    azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
```

## Examples

### Self-Hosted Runner Management

A complete workflow that starts a VM, runs jobs on it, then deallocates it:

```yaml
name: CI with Self-Hosted Runner
on: [push, pull_request]

jobs:
  start-runner:
    runs-on: ubuntu-latest
    outputs:
      runner-started: ${{ steps.start.outputs.runner-started }}
    steps:
      - name: Start Runner VM
        id: start
        uses: farooq-teqniqly/azure-vm-control@v1
        with:
          azure_resource_group_name: 'github-runners'
          azure_vm_name: 'ubuntu-runner'
          azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
          operation: 'start'

  test:
    needs: start-runner
    runs-on: self-hosted
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run tests
        run: |
          echo "Running tests on self-hosted runner"

  cleanup:
    needs: [start-runner, test]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Deallocate Runner VM
        uses: farooq-teqniqly/azure-vm-control@v1
        with:
          azure_resource_group_name: 'github-runners'
          azure_vm_name: 'ubuntu-runner'
          azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
          operation: 'deallocate'
```

### Scheduled VM Control

Use with scheduled workflows for cost optimization:

```yaml
name: Daily VM Management
on:
  schedule:
    - cron: '0 9 * * 1-5'  # Start at 9 AM weekdays
    - cron: '0 18 * * 1-5' # Stop at 6 PM weekdays

jobs:
  control-vm:
    runs-on: ubuntu-latest
    steps:
      - name: Control VM
        uses: farooq-teqniqly/azure-vm-control@v1
        with:
          azure_resource_group_name: 'office-hours-runners'
          azure_vm_name: 'dev-runner'
          azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
          operation: ${{ github.event.schedule == '0 9 * * 1-5' && 'start' || 'deallocate' }}
```

### Performance Testing with Cost Optimization

This action is used in real-world projects to automatically start VMs before running performance tests and deallocate them afterward to minimize Azure costs. See these repositories for examples:

- [**tq-results**](https://github.com/farooq-teqniqly/tq-results) - Performance testing results and analysis
- [**tq-sluggo**](https://github.com/farooq-teqniqly/tq-sluggo) - Performance testing framework

Example workflow pattern:

```yaml
name: Performance Tests
on: [workflow_dispatch]

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - name: Start Performance Test VM
        uses: farooq-teqniqly/azure-vm-control@v1
        with:
          azure_resource_group_name: 'perf-test-rg'
          azure_vm_name: 'perf-runner'
          azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
          operation: 'start'

  run-tests:
    needs: setup
    runs-on: self-hosted
    steps:
      - name: Run Performance Tests
        run: |
          # Performance test commands here

  cleanup:
    needs: [setup, run-tests]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Deallocate Test VM
        uses: farooq-teqniqly/azure-vm-control@v1
        with:
          azure_resource_group_name: 'perf-test-rg'
          azure_vm_name: 'perf-runner'
          azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
          operation: 'deallocate'
```

## API Reference

### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `azure_resource_group_name` | Azure resource group containing the VM | Yes | - |
| `azure_vm_name` | Name of the Azure Virtual Machine | Yes | - |
| `azure_credentials` | Azure service principal credentials JSON (for credential-based auth) | No* | - |
| `client_id` | Azure client ID for OIDC authentication | No* | - |
| `tenant_id` | Azure tenant ID for OIDC authentication | No* | - |
| `subscription_id` | Azure subscription ID for OIDC authentication | No* | - |
| `operation` | VM operation: `start` or `deallocate` (blocking operations that wait for completion; may take several minutes with no configurable timeout) | No | `start` |

*Either `azure_credentials` or all three OIDC inputs (`client_id`, `tenant_id`, `subscription_id`) must be provided.

### Outputs

This action does not produce outputs but logs the final VM power state.

### Permissions

The Azure service principal needs the following permissions:

- `Microsoft.Compute/virtualMachines/read`
- `Microsoft.Compute/virtualMachines/start/action`
- `Microsoft.Compute/virtualMachines/deallocate/action`

## Benefits

- **Cost Optimization**: Automatically stop VMs when not needed to reduce Azure costs
- **Simplified CI/CD**: Integrate VM lifecycle management directly into GitHub workflows
- **Reliable Operations**: Built-in validation and error handling
- **State-Aware**: Checks current VM state before performing operations
- **Secure**: Uses Azure's official authentication mechanisms
- **Lightweight**: No external dependencies beyond Azure CLI and the official Azure Login action (azure/login)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
