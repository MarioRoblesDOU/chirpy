---
title: Terraform and GitHub Actions
date: 2023-05-03 11:02:00 +/-TTTT
categories: [IaC & CM, Terraform, CI/CD, Tutorials]
tags: [CI/CD, Tutorials] 
img_path: /assets/img/    # TAG names should always be lowercase
---

> THIS IS A WORK IN PROGRESS
{: .prompt-warning }

# **Deploying Terraform in Azure using GitHub Actions Step by Step**

**GitHub Actions** is a CI/CD (continuous integration and continuous delivery) platform that allows us to automate our build, test, and deployment pipeline right in our repository.

In this story, we will learn how to set up **GitHub Actions** to deploy **Terraform** code in **Azure**.

# **1. Prerequisites**

This is the list of prerequisites required to create a DevOps pipeline:

- **Azure Subscription:** If you don’t have an Azure subscription, create a free account at [https://azure.microsoft.com](https://azure.microsoft.com/) before you start.
- **Azure Service Principal (SPN):** is an identity used to authenticate to Azure. See below (Point #2) for instructions to create one.
- **Azure Remote Backend for Terraform:** we will store our Terraform state file in a remote backend location. We will need a Resource Group, Azure Storage Account, and a Container.
- **GitHub Account and GitHub Repository:** we need a GitHub Account to create the GitHub Repository and GitHub Actions.

# **2. Prerequisite: Creating an Azure Service Principal**

Using a **Service Principal**, also known as **SPN**, is a best practice for DevOps or CI/CD environments.

First, we need to authenticate to Azure. To authenticate using **Azure CLI**, we type:

```bash
az login
```

The process will launch the browser, and we will be ready to go after the authentication is complete.

Then create the principal service account using the following command:

```bash
az ad sp create-for-rbac -n somename --role="Contributor" --scopes="/subscriptions/SUBSCRIPTION_ID"
```

This is the result:

```bash
{
  "appId": "123",
  "displayName": "somename",
  "password": "123",
  "tenant": "123"
}
```

These values will be mapped to the Terraform variables:

- **appId** (Azure) → **client_id** (Terraform).
- **password** (Azure) → **client_secret** (Terraform).
- **tenant** (Azure) → **tenant_id** (Terraform).

# 3**. Creating Secrets for Azure**

In our new GitHub repository, we click on the **Settings** menu. Then, expand the **Secrets** left menu and select **Actions**.

Then click on the **New Repository Secret** button to create the secrets.

> Note: creating secrets at the organizational level will make your life easier if you have an organization.
> 

![Secrets](ga_secrets.jpg)

Enter the **Name** and **Secret**, and click on the **Add Secret** button.

> Note: Do not enter any space or press enter after the Secret, or it will break the pipeline execution!
> 

After we enter all secrets, we are ready to go.

# **6) Updating the Terraform Provider**

Now, we will set the backend on the Terraform Provider to store our Terraform state file in the Azure Storage Account.

```hcl
terraform {
  required_version = ">=0.12"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 3.7.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "your-rg"
    storage_account_name = "yoursa"
    container_name       = "yourcn"
    key                  = "terraform.tfstate"
  }
}
```

# **7) Creating a GitHub Actions Workflow for Terraform**

```yaml
name: 'Terraform Plan/Apply'

on:
  push:
    branches:
    - main
  pull_request:
    branches:
    - main
defaults:
  run:
    working-directory: terraform

#Special permissions required for OIDC authentication
permissions:
  id-token: write
  contents: read
  pull-requests: write

#These environment variables are used by the terraform azure provider to setup OIDD authenticate. 
env:
  ARM_CLIENT_ID: "${{ secrets.CLIENT_ID }}"
  ARM_CLIENT_SECRET: "${{ secrets.CLIENT_SECRET }}"
  ARM_SUBSCRIPTION_ID: "${{ secrets.SUBSCRIPTION_ID }}"
  ARM_TENANT_ID: "${{ secrets.TENANT_ID }}"

jobs:                
  terraform-apply:
    name: 'Terraform Apply'
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    # Checkout the repository to the GitHub Actions runner
    - name: Checkout
      uses: actions/checkout@v3

    # Install the latest version of Terraform CLI and configure the Terraform CLI configuration file with a Terraform Cloud user API token
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2

    # Initialize a new or existing Terraform working directory by creating initial files, loading any remote state, downloading modules, etc.
    - name: Terraform Init
      run: terraform init

    # Download saved plan from artifacts  
    - name: Download Terraform Plan
      run: terraform plan -no-color
      continue-on-error: true

    # Terraform Apply
    - name: Terraform Apply
      run: terraform apply --auto-approve
```

> This workflow uses the hashicorp/setup-terraform action to build the infrastructure, to know more about is usage and more examples, you can visit the following links: [https://github.com/marketplace/actions/hashicorp-setup-terraform](https://github.com/marketplace/actions/hashicorp-setup-terraform)  and   https://github.com/Azure-Samples/terraform-github-actions
>