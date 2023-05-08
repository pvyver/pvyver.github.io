---
layout: post
title: Deploying Terraform using Github Actions with OpenID Connect (OIDC)
date:   2023-05-08 
logo: 'github'
comments: true
---
 
![Github Actions](/_images/2023-05-08-github-terraform-actions-deployment_github-actions.png)


## Introduction
Github uses a nice feature called `OpenID Connect (OIDC)` for Federated Authentication.

OpenID Connect (OIDC) allows your GitHub Actions workflows to access resources in Azure, without needing to store the Azure credentials as long-lived GitHub secrets.


For more information on how to setup  you can read the GitHub documentation: [Configuring OpenID Connect in Azure] 


[Configuring OpenID Connect in Azure]:https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure


Authentication is then done with the [azure/login@v1] action.


[azure/login@v1]:https://github.com/Azure/login

``` yaml
    - name: 'Az CLI login'
      uses: azure/login@v1
      with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Possible Terraform issue
This is working fine for direct deployments with ARM or Bicep templates, however, when using the 
When tying to use the authentication with Terraform whith a    `Terraform plan` or a `terraform apply`, you might bump into the following error:

> Error: Error building AzureRM Client: Authenticating using the Azure CLI is only supported as a User (not a Service Principal).
> To authenticate to Azure using a Service Principal, you can use the separate 'Authenticate using a Service Principal'
auth method - instructions for which can be found here: https://www.terraform.io/docs/providers/azurerm/guides/service_principal_client_secret.html



## To enable Terraform to use `OpenID Connect (OIDC)`

### Add the `use_oidc = true` section to the backend ttings as exmplated in the [azurerm] documentation

[azurerm]:https://developer.hashicorp.com/terraform/language/settings/backends/azurerm 

Example `azure.tf`:

``` json
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "3.30.0"
    }
  }
}

provider "azurerm" {
  use_oidc = true
  features {}
}
```

### Declare the environment variables in the `GitHub workflow`:

``` yaml
env:
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
  ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
```



> You can find a working example on my GitHub [TFdeploy] repo

[TFdeploy]:https://github.com/pvyver/TFdeploy