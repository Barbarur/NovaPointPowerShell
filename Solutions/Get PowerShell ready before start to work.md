#AzureAD #Msol #ExchangeOnline #PnP #PowerShell #SPOService  

<br>

## Use PowerShell as Admin

Although normal PowerShell can be used and it’s fine for single commands, PowerShell ISE provides a script panel that will help to write, read, run and test scripts easily.

Better to run always as admin to ensure you won’t have any issue in case need to install additional modules on the way or having access to specific parts of your device.

<br>

## Install needed modules

Azure Active Directory
```
Install-Module AzureAD
```

Microsoft Online
```
Install-Module -Name MSOnline
```

Exchange Online PowerShell
```
Install-Module -Name ExchangeOnlineManagement
```

[SharePoint Online Management Shell](https://learn.microsoft.com/en-us/powershell/sharepoint/sharepoint-online/connect-sharepoint-online?view=sharepoint-ps)
```
Install-Module -Name Microsoft.Online.SharePoint.PowerShell
```

[PnP PowerShell](https://github.com/pnp/powershell)
```
Install-Module -Name PnP.PowerShell -RequiredVersion 1.12.0 -Force
```

```
Install-Module -Name PnP.PowerShell
```

<br>

## Allow your device to run PowerShell scripts

```
Set-ExecutionPolicy Unrestricted -Scope CurrentUser
```

<br>

## Update old modules

```
Update-Module -Name XXX
```

<br>

## Uninstall unnecessary or conflicting modules

```
Uninstall-Module -Name XXX
```

<br>

## Restart PowerShell ISE

Although not always necessary, it’s highly recommended to close the open again PowerShell ISE after each big change you do on the modules.


### Reference
[Get-Credential (Microsoft.PowerShell.Security) - PowerShell | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-credential?view=powershell-7.3&viewFallbackFrom=powershell-7.1)
[Get started with the SharePoint Online Management Shell | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/sharepoint/sharepoint-online/connect-sharepoint-online?view=sharepoint-ps)
[PnP PowerShell Overview | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/sharepoint/sharepoint-pnp/sharepoint-pnp-cmdlets?view=sharepoint-ps)
[Connect to Exchange Online PowerShell | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/exchange/connect-to-exchange-online-powershell?view=exchange-ps)
