---
title: Azure Key Vault managed storage account - PowerShell version
description: The managed storage account feature provides a seamless integration, between Azure Key Vault and an Azure storage account.
ms.topic: tutorial
ms.service: azure-key-vault
ms.subservice: secrets
author: msmbaldwin
ms.author: mbaldwin
ms.date: 04/14/2025

ms.custom: devx-track-azurepowershell

# Customer intent: As a developer I want storage credentials and SAS tokens to be managed securely by Azure Key Vault.
---

# Manage storage account keys with Key Vault and Azure PowerShell (legacy)

> [!IMPORTANT]
> Key Vault Managed Storage Account Keys (legacy) is supported as-is with no more updates planned. Only Account SAS are supported with SAS definitions signed storage service version no later than 2018-03-28.

> [!IMPORTANT]
> We recommend using Azure Storage integration with Microsoft Entra ID, Microsoft's cloud-based identity and access management service. Microsoft Entra integration is available for [Azure blobs and queues](/azure/storage/blobs/authorize-access-azure-active-directory), and provides OAuth2 token-based access to Azure Storage (just like Azure Key Vault).
> Microsoft Entra ID allows you to authenticate your client application by using an application or user identity, instead of storage account credentials. You can use an [Microsoft Entra managed identity](/azure/active-directory/managed-identities-azure-resources/) when you run on Azure. Managed identities remove the need for client authentication and storing credentials in or with your application. Use this solution only when Microsoft Entra authentication is not possible.

An Azure storage account uses credentials comprising an account name and a key. The key is autogenerated and serves as a password, rather than an as a cryptographic key. Key Vault manages storage account keys by periodically regenerating them in storage account and provides shared access signature tokens for delegated access to resources in your storage account.

You can use the Key Vault managed storage account key feature to list (sync) keys with an Azure storage account, and regenerate (rotate) the keys periodically. You can manage keys for both storage accounts and Classic storage accounts.

When you use the managed storage account key feature, consider the following points:

- Key values are never returned in response to a caller.
- Only Key Vault should manage your storage account keys. Don't manage the keys yourself and avoid interfering with Key Vault processes.
- Only a single Key Vault object should manage storage account keys. Don't allow key management from multiple objects.
- Regenerate keys by using Key Vault only. Don't manually regenerate your storage account keys.

> [!IMPORTANT]
> Regenerating key directly in storage account breaks managed storage account setup and can invalidate SAS tokens in use and cause an outage.

[!INCLUDE [updated-for-az](~/reusable-content/ce-skilling/azure/includes/updated-for-az.md)]

## Service principal application ID

A Microsoft Entra tenant provides each registered application with a [service principal](/azure/active-directory/develop/developer-glossary#service-principal-object). The service principal serves as the application ID, which is used during authorization setup for access to other Azure resources via Azure RBAC.

Key Vault is a Microsoft application that's pre-registered in all Microsoft Entra tenants. Key Vault is registered under the same Application ID in each Azure cloud.

| Tenants | Cloud | Application ID |
| --- | --- | --- |
| Microsoft Entra ID | Azure Government | `7e7c393b-45d0-48b1-a35e-2905ddf8183c` |
| Microsoft Entra ID | Azure public | `cfa8b339-82a2-471a-a3c9-0fc0be7a4093` |
| Other  | Any | `cfa8b339-82a2-471a-a3c9-0fc0be7a4093` |

## Prerequisites

To complete this guide, you must first do the following:

- [Install the Azure PowerShell module](/powershell/azure/install-azure-powershell).
- [Create a key vault](quick-create-powershell.md)
- [Create an Azure storage account](/azure/storage/common/storage-account-create?tabs=azure-powershell). The storage account name must use only lowercase letters and numbers. The length of the name must be between 3 and 24 characters.

## Manage storage account keys

### Connect to your Azure account

Authenticate your PowerShell session using the [Connect-AzAccount](/powershell/module/az.accounts/connect-azaccount) cmdlet.

```azurepowershell-interactive
Connect-AzAccount
```

If you have multiple Azure subscriptions, you can list them using the [Get-AzSubscription](/powershell/module/az.accounts/get-azsubscription) cmdlet, and specify the subscription you wish to use with the [Set-AzContext](/powershell/module/az.accounts/set-azcontext) cmdlet.

```azurepowershell-interactive
Set-AzContext -SubscriptionId <subscriptionId>
```

### Set variables

First, set the variables to be used by the PowerShell cmdlets in the following steps. Be sure to update the "YourResourceGroupName", "YourStorageAccountName", and "YourKeyVaultName" placeholders, and set $keyVaultSpAppId to `cfa8b339-82a2-471a-a3c9-0fc0be7a4093` (as specified in [Service principal application ID](#service-principal-application-id)).

We'll also use the Azure PowerShell [Get-AzContext](/powershell/module/az.accounts/get-azcontext) and [Get-AzStorageAccount](/powershell/module/az.storage/get-azstorageaccount) cmdlets to get your user ID and the context of your Azure storage account.

```azurepowershell-interactive
$resourceGroupName = <YourResourceGroupName>
$storageAccountName = <YourStorageAccountName>
$keyVaultName = <YourKeyVaultName>
$keyVaultSpAppId = "cfa8b339-82a2-471a-a3c9-0fc0be7a4093"
$storageAccountKey = "key1" #(key1 or key2 are allowed)

# Get your User Id
$userId = (Get-AzContext).Account.Id

# Get a reference to your Azure storage account
$storageAccount = Get-AzStorageAccount -ResourceGroupName $resourceGroupName -StorageAccountName $storageAccountName

```
>[!Note]
> For Classic Storage Account use "primary" and "secondary" for $storageAccountKey <br>
> Use 'Get-AzResource -Name "ClassicStorageAccountName" -ResourceGroupName $resourceGroupName' instead of'Get-AzStorageAccount' for Classic Storage Account

### Give Key Vault access to your storage account

Before Key Vault can access and manage your storage account keys, you must authorize its access your storage account. The Key Vault application requires permissions to *list* and *regenerate* keys for your storage account. These permissions are enabled through the Azure built-in role [Storage Account Key Operator Service Role](/azure/role-based-access-control/built-in-roles#storage-account-key-operator-service-role).

Assign this role to the Key Vault service principal, limiting scope to your storage account, using the Azure PowerShell [New-AzRoleAssignment](/powershell/module/az.resources/new-azroleassignment) cmdlet.

```azurepowershell-interactive
# Assign Azure role "Storage Account Key Operator Service Role" to Key Vault, limiting the access scope to your storage account. For a classic storage account, use "Classic Storage Account Key Operator Service Role."
New-AzRoleAssignment -ApplicationId $keyVaultSpAppId -RoleDefinitionName 'Storage Account Key Operator Service Role' -Scope $storageAccount.Id
```

Upon successful role assignment, you should see output similar to the following example:

```console
RoleAssignmentId   : /subscriptions/aaaa0a0a-bb1b-cc2c-dd3d-eeeeee4e4e4e/resourceGroups/rgContoso/providers/Microsoft.Storage/storageAccounts/sacontoso/providers/Microsoft.Authorization/roleAssignments/00000000-0000-0000-0000-000000000000
Scope              : /subscriptions/aaaa0a0a-bb1b-cc2c-dd3d-eeeeee4e4e4e/resourceGroups/rgContoso/providers/Microsoft.Storage/storageAccounts/sacontoso
DisplayName        : Azure Key Vault
SignInName         :
RoleDefinitionName : storage account Key Operator Service Role
RoleDefinitionId   : 81a9662b-bebf-436f-a333-f67b29880f12
ObjectId           : aaaaaaaa-0000-1111-2222-bbbbbbbbbbbb
ObjectType         : ServicePrincipal
CanDelegate        : False
```

If Key Vault has already been added to the role on your storage account, you'll receive a *"The role assignment already exists."* error. You can also verify the role assignment, using the storage account "Access control (IAM)" page in the Azure portal.

### Give your user account permission to managed storage accounts

Use the Azure PowerShell [Set-AzKeyVaultAccessPolicy](/powershell/module/az.keyvault/set-azkeyvaultaccesspolicy) cmdlet to update the Key Vault access policy and grant storage account permissions to your user account.

```azurepowershell-interactive
# Give your user principal access to all storage account permissions, on your Key Vault instance

Set-AzKeyVaultAccessPolicy -VaultName $keyVaultName -UserPrincipalName $userId -PermissionsToStorage get, list, delete, set, update, regeneratekey, getsas, listsas, deletesas, setsas, recover, backup, restore, purge
```

The permissions for storage accounts aren't available on the storage account "Access policies" page in the Azure portal.

### Add a managed storage account to your Key Vault instance

Use the Azure PowerShell [Add-AzKeyVaultManagedStorageAccount](/powershell/module/az.keyvault/add-azkeyvaultmanagedstorageaccount) cmdlet to create a managed storage account in your Key Vault instance. The  `-DisableAutoRegenerateKey` switch specifies NOT to regenerate the storage account keys.

```azurepowershell-interactive
# Add your storage account to your Key Vault's managed storage accounts

Add-AzKeyVaultManagedStorageAccount -VaultName $keyVaultName -AccountName $storageAccountName -AccountResourceId $storageAccount.Id -ActiveKeyName $storageAccountKey -DisableAutoRegenerateKey
```

Upon successful addition of the storage account with no key regeneration, you should see output similar to the following example:

```console
Id                  : https://kvcontoso.vault.azure.net:443/storage/sacontoso
Vault Name          : kvcontoso
AccountName         : sacontoso
Account Resource Id : /subscriptions/aaaa0a0a-bb1b-cc2c-dd3d-eeeeee4e4e4e/resourceGroups/rgContoso/providers/Microsoft.Storage/storageAccounts/sacontoso
Active Key Name     : key1
Auto Regenerate Key : False
Regeneration Period : 90.00:00:00
Enabled             : True
Created             : 11/19/2018 11:54:47 PM
Updated             : 11/19/2018 11:54:47 PM
Tags                :
```

### Enable key regeneration

If you want Key Vault to regenerate your storage account keys periodically, you can use the Azure PowerShell [Add-AzKeyVaultManagedStorageAccount](/powershell/module/az.keyvault/add-azkeyvaultmanagedstorageaccount) cmdlet to set a regeneration period. In this example, we set a regeneration period of 30 days. When it's time to rotate, Key Vault regenerates the inactive key and then sets the newly created key as active. The key used to issue SAS tokens is the active key.

```azurepowershell-interactive
$regenPeriod = [System.Timespan]::FromDays(30)

Add-AzKeyVaultManagedStorageAccount -VaultName $keyVaultName -AccountName $storageAccountName -AccountResourceId $storageAccount.Id -ActiveKeyName $storageAccountKey -RegenerationPeriod $regenPeriod
```

Upon successful addition of the storage account with key regeneration, you should see output similar to the following example:

```console
Id                  : https://kvcontoso.vault.azure.net:443/storage/sacontoso
Vault Name          : kvcontoso
AccountName         : sacontoso
Account Resource Id : /subscriptions/aaaa0a0a-bb1b-cc2c-dd3d-eeeeee4e4e4e/resourceGroups/rgContoso/providers/Microsoft.Storage/storageAccounts/sacontoso
Active Key Name     : key1
Auto Regenerate Key : True
Regeneration Period : 30.00:00:00
Enabled             : True
Created             : 11/19/2018 11:54:47 PM
Updated             : 11/19/2018 11:54:47 PM
Tags                :
```

## Shared access signature tokens

You can also ask Key Vault to generate shared access signature tokens. A shared access signature provides delegated access to resources in your storage account. You can grant clients access to resources in your storage account without sharing your account keys. A shared access signature provides you with a secure way to share your storage resources without compromising your account keys.

The commands in this section complete the following actions:

- Set an account shared access signature definition.
- Set a Key Vault managed storage shared access signature definition in the vault. The definition has the template URI of the shared access signature token that was created. The definition has the shared access signature type `account` and is valid for N days.
- Verify that the shared access signature was saved in your key vault as a secret.

### Set variables

First, set the variables to be used by the PowerShell cmdlets in the following steps. Be sure to update the \<YourStorageAccountName\> and \<YourKeyVaultName\> placeholders.

```azurepowershell-interactive
$storageAccountName = <YourStorageAccountName>
$keyVaultName = <YourKeyVaultName>
```

### Define a shared access signature definition template

Key Vault uses SAS definition template to generate tokens for client applications. 

SAS definition template example:
```azurepowershell-interactive
$sasTemplate="sv=2018-03-28&ss=bfqt&srt=sco&sp=rw&spr=https"
```

#### Account SAS parameters required in SAS definition template for Key Vault
|SAS Query Parameter|Description|  
|-------------------------|-----------------|  
|`SignedVersion (sv)`|Required. Specifies the signed storage service version to use to authorize requests made with this account SAS. Must be set to version 2015-04-05 or later. **Key Vault supports versions no later than 2018-03-28**|  
|`SignedServices (ss)`|Required. Specifies the signed services accessible with the account SAS. Possible values include:<br /><br /> - Blob (`b`)<br />- Queue (`q`)<br />- Table (`t`)<br />- File (`f`)<br /><br /> You can combine values to provide access to more than one service. For example, `ss=bf` specifies access to the Blob and File endpoints.|  
|`SignedResourceTypes (srt)`|Required. Specifies the signed resource types that are accessible with the account SAS.<br /><br /> - Service (`s`): Access to service-level APIs (*for example*, Get/Set Service Properties, Get Service Stats, List Containers/Queues/Tables/Shares)<br />- Container (`c`): Access to container-level APIs (*for example*, Create/Delete Container, Create/Delete Queue, Create/Delete Table, Create/Delete Share, List Blobs/Files and Directories)<br />- Object (`o`): Access to object-level APIs for  blobs, queue messages,  table entities, and files(*for example,* Put Blob, Query Entity, Get Messages, Create File, etc.)<br /><br /> You can combine values to provide access to more than one resource type. For example, `srt=sc` specifies access to service and container resources.|  
|`SignedPermission (sp)`|Required. Specifies the signed permissions for the account SAS. Permissions are only valid if they match the specified signed resource type; otherwise they are ignored.<br /><br /> - Read (`r`): Valid for all signed resources types (Service, Container, and Object). Permits read permissions to the specified resource type.<br />- Write (`w`): Valid for all signed resources types (Service, Container, and Object). Permits write permissions to the specified resource type.<br />- Delete (`d`): Valid for Container and Object resource types, except for queue messages.<br />- Permanent Delete (`y`): Valid for Object resource type of Blob only.<br />- List (`l`): Valid for Service and Container resource types only.<br />- Add (`a`): Valid for the following Object resource types only: queue messages, table entities, and append blobs.<br />- Create (`c`): Valid for the following Object resource types only: blobs and files. Users can create new blobs or files, but may not overwrite existing blobs or files.<br />- Update (`u`): Valid for the following Object resource types only: queue messages and table entities.<br />- Process (`p`): Valid for the following Object resource type only: queue messages.<br/>- Tag (`t`): Valid for the following Object resource type only: blobs. Permits blob tag operations.<br/>- Filter (`f`): Valid for the following Object resource type only: blob. Permits filtering by blob tag.<br/>- Set Immutability Policy (`i`): Valid for the following Object resource type only: blob. Permits set/delete immutability policy and legal hold on a blob.|
|`SignedProtocol (spr)`|Optional. Specifies the protocol permitted for a request made with the account SAS. Possible values are both HTTPS and HTTP (`https,http`) or HTTPS only (`https`).  The default value is `https,http`.<br /><br /> HTTP only is not a permitted value.|    

For more information about account SAS, see:
[Create an account SAS](/rest/api/storageservices/create-account-sas)

> [!NOTE]
> Key Vault ignores lifetime parameters like 'Signed Expiry', 'Signed Start' and parameters introduced after 2018-03-28 version

### Set shared access signature definition in Key Vault

Use the Azure PowerShell [Set-AzKeyVaultManagedStorageSasDefinition](/powershell/module/az.keyvault/set-azkeyvaultmanagedstoragesasdefinition) cmdlet to create a shared access signature definition.  You can provide the name of your choice to the `-Name` parameter.

```azurepowershell-interactive
Set-AzKeyVaultManagedStorageSasDefinition -AccountName $storageAccountName -VaultName $keyVaultName -Name <YourSASDefinitionName> -TemplateUri $sasTemplate -SasType 'account' -ValidityPeriod ([System.Timespan]::FromDays(1))
```

### Verify the shared access signature definition

You can verify that the shared access signature definition has been stored in your key vault using the Azure PowerShell [Get-AzKeyVaultSecret](/powershell/module/az.keyvault/get-azkeyvaultsecret) cmdlet.

First, find the shared access signature definition in your key vault.

```azurepowershell-interactive
Get-AzKeyVaultSecret -VaultName <YourKeyVaultName>
```

The secret corresponding to your SAS definition will have these properties:

```console
Vault Name   : <YourKeyVaultName>
Name         : <SecretName>
...
Content Type : application/vnd.ms-sastoken-storage
Tags         :
```

You can now use the [Get-AzKeyVaultSecret](/powershell/module/az.keyvault/get-azkeyvaultsecret) cmdlet with the `VaultName` and `Name` parameters to view the contents of that secret.

```azurepowershell-interactive
$secretValueText = Get-AzKeyVaultSecret -VaultName <YourKeyVaultName> -Name <SecretName> -AsPlainText
Write-Output $secretValueText
```

The output of this command will show your SAS definition string.

## Next steps

- [Managed storage account key samples](https://github.com/Azure-Samples?utf8=%E2%9C%93&q=key+vault+storage&type=&language=)
- [Key Vault PowerShell reference](/powershell/module/az.keyvault/#key_vault)
