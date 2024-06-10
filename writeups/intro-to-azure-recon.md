---
layout: default
---
# Overview

This is a writeup for the [Intro to Azure Recon with BloodHound](https://pwnedlabs.io/labs/intro-to-azure-recon-with-bloodhound) lab at pwned labs

I worked through the walkthrough last week as I hadn't used bloodhound before, now I'm going to run through again while trying to write some re-usable scripts for future labs

# Lab

## Bloodhound

I'm using WSL2 kali linux, lets follow the setup guide for bloodhound [here](https://bloodhound.readthedocs.io/en/latest/installation/linux.html) and then run the neo4j console

![neo4j console](/images/azure-recon-neo4j.png)

Checking the azurehound docs [here](https://github.com/BloodHoundAD/AzureHound?tab=readme-ov-file#quickstart) we need to run 

```bash
azurehound list -u "$USERNAME" -p "$PASSWORD" -t "$TENANT" -o "mytenant.json"
```

We can get the tenant ID from a simple az login using the lab provided creds

![az login](/images/azure-recon-az-login.png)

Great, now we can run azurehound with our details

![azurehound](/images/azure-recon-azurehound.png)

Now we select upload data and provide our json output

![import](/images/azure-recon-upload.png)

Lets seach for the provided user to check the import was sucessful

![jose](/images/azure-recon-jose.png)

We can select the "Azure AD Admin Roles" section to view what roles our user has (I also changed the view because my eyes were burning)

![admin roles](/images/azure-recon-adminroles.png)

The script provided in the walkthrough for security attributes also works for larger environments, so lets look at role enumeration

## Role Enumeration

Lets write a generic script for enumerating permissions

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
    Set-AzContext -Tenant $tenant | Out-Null
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

## VM Userdata

The only role we have is for the VM, so lets write a script that would also work in a larger environment
Userdata is only returned when directly specifying an individual VM

```powershell
#Initialise list variables incase we only get one result
$scopes = @()
$tenants = @()
$roles = @()
$VMs = @()

#Authenticate to azure
$session = Connect-AzAccount -UseDeviceAuthentication

#Get all tenants for this user
$tenants += (Get-AzTenant).Id

#Add subscriptions
foreach($tenant in $tenants) {
    Set-AzContext -Tenant $tenant | Out-Null
    $scopes += (Get-AzSubscription -TenantId $tenant) | Select-Object Id, @{n="TenantId";e={$tenant}}
}

#Enumerate VMs
foreach($scope in $scopes) {
    Set-AzContext -Tenant $scope.TenantId -Subscription $scope.Id | Out-Null
    $VMList = Get-AzVM
    foreach($VM in $VMList) {
        $VMs += Get-AzVM -UserData -ResourceGroupName $VM.ResourceGroupName -Name $VM.Name | Select-Object *, @{n="UserDataString";e={[Text.Encoding]::Utf8.GetString([Convert]::FromBase64String($_.UserData))}}
    }
}

$VMs | Where-Object {$_.UserData -ne $null} | Format-List Name, UserDataString
```

![VM userdata](/images/azure-recon-userdata.png)

I ran this on my home tenant and was happy to see no sketchy userdata 🥳

## Storage Account Enumeration

Lets rerun our role enumeration script using the new creds we pulled from user data

![Security account roles](/images/azure-recon-security-roles.png)

We only have access to a storage account, lets write a script to enumerate the blobs and download a target

```powershell
#Get the storage account name to enumerate
$storageAccount = Read-Host -Prompt "Enter the storage account name"

#Authenticate to azure
$session = Connect-AzAccount -UseDeviceAuthentication

#Create a storage context
$storageContext = New-AzStorageContext -StorageAccountName $storageAccount -UseConnectedAccount

#Get available storage containers
$storageContainers = Get-AzStorageContainer -Context $storageContext

#Get available blobs from storage containers
$blobs = foreach($container in $storageContainers) {
    Get-AzStorageBlob -Container $container.Name -Context $storageContext
}
$blobs | Format-Table Name, ContentType, LastModified

#Retrieve blob content
$targetBlob = Read-Host -Prompt "Enter the blob name you want to view"
$targetBlobDetails = $blobs | Where-Object {$_.Name -eq $targetBlob}
If ($null -eq $targetBlobDetails) {
    Write-Error "Target blob was not found, please rerun and refer to output"
}
$blobDownload = Get-AzStorageBlobContent -Container $targetBlobDetails.BlobClient.BlobContainerName -Blob $targetBlob -Destination ".\" -Context $storageContext

#Output blob content
Get-Content -Path ".\$($blobDownload.Name)"
```

![Flag!](/images/azure-recon-flag.png)

Nice, we have the flag and I'm lazy so I will stop here

# Lessons Learnt

First lesson is that when doing a write-up, definitely give it a go blind first instead of starting the walkthrough and getting too engrossed... oops
