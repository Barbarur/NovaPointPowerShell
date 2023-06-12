#AzureAD #PowerShell 
 
 <br>
 
## Connect to Azure AD

```powershell
Connect-AzureAD
```


## Restore MS365 Group

```powershell

Identify the MS365 group ID to be restored.

Get-AzureADMSDeletedGroup -All $True

Restore the MS365 group using the group ID.

Restore-AzureADMSDeletedDirectoryObject -ID "GroupID"
```