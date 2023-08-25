---
layout: post
title: 'PowerShell Authentication tokens for Azure REST API'
date:   2023-08-11 
logo: 'fa fa-code'
comments: true
---

### Table of contents

- [Introduction](#introduction)
	- [Azure REST API Authentication](#azure-rest-api-authentication)
		- [JSON Web Token (JWT)](#json-web-token-jwt)
		- [Authentication to the Management API](#authentication-to-the-management-api)
- [Authentication with an App Registration](#authentication-with-an-app-registration)
		- [Example](#example)
- [Authentication with current account](#authentication-with-current-account)
	- [Using Get-AccessToken](#using-get-accesstoken)
		- [Example](#example-1)
	- [Using the token stored in the profile](#using-the-token-stored-in-the-profile)
		- [Example](#example-2)


## Introduction

The [Azure REST API] is a powerful way for interacting with Azure service resources.

[Azure REST API]:https://learn.microsoft.com/en-us/rest/api/azure/

In this article, I want to explain the authentication options that you have with PowerShell to interact with the `Azure REST API`

### Azure REST API Authentication

Authentication to the `Azure REST API` is done by using an [OAuth2] bearer token.

[OAuth2]:https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow

With this token you can authenticate your request on the Azure REST API:

<img src="/_images/2023-08-11-powershell-authentication-token-azure-rest-api-authication-flow.png">

#### JSON Web Token (JWT)

The token itself is a [JSON Web Token (JWT)], it consist of three parts separated by dots (.), which are:

[JSON Web Token (JWT)]:https://jwt.io/introduction

- Header
- Payload
- Signature

Therefore, a JWT typically looks like the following.

```
xxxxx.yyyyy.zzzzz
```

> **Note**:  
> To decode a JWT token, you can use the online tool over here:
> [https://jwt.io/]
> 
> To dump the claims in your bearer token so you can validate their contents you can use the following online tool:
> [https://jwt.ms/]

[https://jwt.io/]:https://jwt.io/
[https://jwt.ms/]:https://jwt.ms/

#### Authentication to the Management API

An Access token is a Bearer token that you will have to add in all request headers to be authenticated.

For example, you might send an `HTTPS GET` request method for an `Azure Resource Manager provider` to [list subscriptions] by using request header fields that are similar to the following:

[list subscriptions]:https://learn.microsoft.com/en-us/rest/api/resources/subscriptions/list?tabs=HTTP

``` powershell
$path = "/subscriptions"
$apiVersion = "2022-12-01"
$token = "<your-acquired-token>"

$authHeader = @{
    'Content-Type'  = 'application/json'
    'Authorization' = "Bearer $token"
}

# invoke Azure Management REST API
$uri = "https://management.azure.com$($path)?api-version=$($apiVersion)"
Write-Host "Invoking request to: '$uri'"
$data = Invoke-RestMethod -Method Get -Uri $uri -Headers $authHeader
```

## Authentication with an App Registration

A convenient way of authentication is using an App Registration.

The access token can be aquired using an [access token request with a shared secret].

[access token request with a shared secret]:https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow#first-case-access-token-request-with-a-shared-secret

`Get-AppRegistrationAuthorizationToken` function:

``` powershell
function Get-AppRegistrationAuthorizationToken {
	[CmdletBinding()]
	param
	(
		[Parameter(Mandatory = $true)]
		[string]$TenantID,
		[Parameter(Mandatory = $true)]
		[string]$ClientID,
		[Parameter(Mandatory = $true)]
		[string]$ClientSecret
	)

	$TokenEndpoint = "https://login.windows.net/$TenantID/oauth2/token"
	$ARMResource = "https://management.core.windows.net/"

	$Body = @{
		'resource'      = $ARMResource
		'client_id'     = $ClientID
		'grant_type'    = 'client_credentials'
		'client_secret' = $ClientSecret
	}

	$params = @{
		ContentType = 'application/x-www-form-urlencoded'
		Headers     = @{'accept' = 'application/json' }
		Body        = $Body
		Method      = 'Post'
		URI         = $TokenEndpoint
	}

	$token = Invoke-RestMethod @params

	Return ($token.access_token).ToString()
}
```

#### Example 

``` powershell
# set variables
$client_id = "<your-app-registration-client-id>"
$app_secret = "<your-app-registration-secret>"
$tenantid = "<your-tenant-id>"
$token = Get-AppRegistrationAuthorizationToken -TenantID $tenantid -ClientID $client_id -ClientSecret $app_secret
$path = "/subscriptions"
$apiVersion = "2022-12-01"
$authHeader = @{
    'Authorization' = "Bearer $($token)"
}

# invoke Azure Management REST API
$uri = "https://management.azure.com$($path)?api-version=$($apiVersion)"
$result = Invoke-RestMethod -Method Get -Uri $uri -Headers $authHeader

# return result
$result.value 

```

## Authentication with current account

When already authenticated, you can get the access token for the identity that is already logged in into Azure.

### Using Get-AccessToken

The PowerShell command [Get-AccessToken] (from the `Az.Accounts` module) can be used to get a fresh access token.

[Get-AccessToken]:https://learn.microsoft.com/en-us/powershell/module/az.accounts/get-azaccesstoken?view=azps-10.2.0

Using the `ResourceTypeName`

```
Get-AzAccessToken
   [-ResourceTypeName <String>]
   [-TenantId <String>]
```

``` powershell
Get-AzAccessToken -ResourceTypeName Arm
```

or 

Using the `ResourceUrl`

```
Get-AzAccessToken
   -ResourceUrl <String>
   [-TenantId <String>]
```

``` powershell
Get-AzAccessToken -ResourceUrl "https://management.core.windows.net/"
```

#### Example 

``` powershell
# set variables
$token = Get-AzAccessToken -ResourceUrl "https://management.core.windows.net/"
$path = "/subscriptions"
$apiVersion = "2022-12-01"
$authHeader = @{
    'Authorization' = "Bearer $($token.Token)"
}

# invoke Azure Management REST API
$uri = "https://management.azure.com$($path)?api-version=$($apiVersion)"
$result = Invoke-RestMethod -Method Get -Uri $uri -Headers $authHeader

# return result
$result.value 
```

### Using the token stored in the profile

Using the `Microsoft.Azure.Commands.ResourceManager.Common Namespace` .NET Namespace, you can use the `AcquireAccessToken` Method to get the stored token in profile.

`Get-AzureCachedAccessToken` function:

``` powershell
function Get-AzureCachedAccessToken {
    # get current Azure Profile
	$azProfile = [Microsoft.Azure.Commands.Common.Authentication.Abstractions.AzureRmProfileProvider]::Instance.Profile

    # gets the metadata used to authenticate Azure Resource Manager requests.
	$currentAzureContext = Get-AzContext

    # initiate client session to access profile
	$profileClient = New-Object Microsoft.Azure.Commands.ResourceManager.Common.RMProfileClient($azProfile)

    # get the token from the profile
	$token = $profileClient.AcquireAccessToken($currentAzureContext.Tenant.TenantId)
	return $token.AccessToken
}
```

#### Example 

``` powershell
$token = Get-AzureCachedAccessToken

$path = "/subscriptions"
$apiVersion = "2022-12-01"
$authHeader = @{
    'Authorization' = "Bearer $($token)"
}

# invoke Azure Management REST API
$uri = "https://management.azure.com$($path)?api-version=$($apiVersion)"
$result = Invoke-RestMethod -Method Get -Uri $uri -Headers $authHeader

$result.value 

```
