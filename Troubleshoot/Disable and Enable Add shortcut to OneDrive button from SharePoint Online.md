#Troubleshoot #SiteCollection #Subsite #OneDrive #PowerShell 

<br>

## Disable

```powershell
#Define Parameters
$AdminSiteURL = "https://{DOMAIN}-admin.sharepoint.com"

#Get Credentials
$Cred  = Get-Credential

#Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL –Credential $Cred

#Disabling “Add to shortcut to OneDrive” button 
Set-SPOTenant -DisableAddShortCutsToOneDrive $True
```

<br>

## Enable

```powershell
#Define Parameters
$AdminSiteURL = "https://{DOMAIN}-admin.sharepoint.com"

#Get Credentials
$Cred  = Get-Credential

#Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL –Credential $Cred

#Disabling “Add to shortcut to OneDrive” button 
Set-SPOTenant -DisableAddShortCutsToOneDrive $False
```