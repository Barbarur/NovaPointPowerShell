#ErrorMessage #PowerShell #Troubleshoot #HubSite

## Error Disable

*This site has exceeded its maximum file storage limit. To free up space, delete files you don't need and empty the recycle bin.*

## Solution

If tenant and site have enough storage available, this might be due lack of storage at ***-my*** root site, i.e. *https://<Domain>-my.sharepoint.com/*

Increase site storage using PowerShell using the command below measure in Mbs:

```powershell
Set-SPOSite -Identity https://<Domain>-my.sharepoint.com/ -StorageQuota 26214400
```