#Automation #PowerShell #PnP #SiteCollection #Subsite #MicrosoftSearch

<br>

## Remove Microsoft Search bar

```powershell
#Define Parameters
$SiteURL = "https://DOMAIN.sharepoint.com/sites/SITENAME"

#Connect to SharePoint Site
Connect-PnPOnline -Url $SiteURL -UseWebLogin

#Hide Search bard command
Set-PnPSearchSettings -SearchBoxInNavBar Hidden -Scope Web

Write-host -f Green "Done"
```

<br>

## Restore Microsoft Search bar

```powershell
#Define Parameters
$SiteURL = "https://DOMAIN.sharepoint.com/sites/SITENAME"

#Connect to SharePoint Site
Connect-PnPOnline -Url $SiteURL -UseWebLogin

#Show Search bard command
Set-PnPSearchSettings -SearchBoxInNavBar AllPages -Scope Web
Set-PnPSearchSettings -SearchBoxInNavBar AllPages -Scope Site

Write-host -f Green "Done"
```