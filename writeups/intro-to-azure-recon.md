---
layout: default
---

# Overview

This is a writeup for the [Intro to Azure Recon with BloodHound](https://pwnedlabs.io/labs/intro-to-azure-recon-with-bloodhound) lab at pwned labs

I worked through the walkthrough last week as I hadn't used bloodhound before, now I'm going to run through again while trying to write some re-usable scripts for future labs

# Lab

To start with we are given some user credentials for azure, lets write a generic script for enumerating permissions

```powershell
$session = Connect-AzAccount -UseDeviceAuthentication
$roles = Get-AzRoleAssignment -SignInName $session.Context.Account.Id -ExpandPrincipalGroups
$roles | Format-Table RoleDefinitionName, Scope, CanDelegate
```

![That was quick](/images/azure-recon-roles.png)

OK that works for a single sub, but lets future proof it by adding support for multiple subscriptions and tenants

```powershell
#Initialise list variables incase we only get one result
$scopes = @()
$tenants = @()
$roles = @()

#Authenticate to azure
$session = Connect-AzAccount -UseDeviceAuthentication

#Get all tenants for this user
$tenants += (Get-AzTenant).Id

#Add subscriptions
foreach($tenant in $tenants) {
    Set-AzContext -Tenant $tenant
    $scopes += (Get-AzSubscription -TenantId $tenant) | Select-Object Id, @{n="TenantId";e={$tenant}}
}

#Enumerate roles
foreach($scope in $scopes) {
    Set-AzContext -Tenant $scope.TenantId -Subscription $scope.Id | Out-Null
    $roles += Get-AzRoleAssignment -SignInName $session.Context.Account.Id -ExpandPrincipalGroups -IncludeClassicAdministrators
}

$roles | Format-Table RoleDefinitionName, Scope, CanDelegate
```

This was interestingly awkward, for some reason the Get-AzRoleAssignment cmdlet doesn't allow you to specify a scope and sign-in name when using expand principal groups

I got around this by just setting the context and appending the reults for each subscription, I'll keep this around for future Azure labs

# Lessons Learnt

First lesson is that when doing a write-up, definitely give it a go blind first instead of starting the walkthrough and getting too engrossed... oops
