#Automation #SharePointOnline #PowerShell #PnP #SiteSettings

<br>

```powershell
#Define Parameters
$AdminSiteURL="https://{DOMAIN}-admin.sharepoint.com"
$TimezoneName = "(UTC+10:00) Canberra, Melbourne, Sydney"
$localeIDnumber = "3081" #English (Australia)

#Connect to PnP Online
Connect-PnPOnline -Url $AdminSiteURL -UseWebLogin
  
#Get All Sites
$AllSites = Get-PnPTenantSite -IncludeOneDriveSites 

#Loop through each site
ForEach($Site in $AllSites)
{ 
    Write-Host -f Yellow "Processing Site: "$Site.URL

    #Connect to OneDrive for Business Site
    Connect-PnPOnline $Site.URL -Credentials $Cred
  
    #Get the Web
    $web = Get-PnPWeb -Includes RegionalSettings.TimeZones
  
    #Get the time zone
    $Timezone  = $Web.RegionalSettings.TimeZones | Where {$_.Description -eq $TimezoneName}
  
    If($Timezone -ne $Null)
    {
        #Update time zone of the site
        $Web.RegionalSettings.LocaleId = $localeIDnumber
        $Web.RegionalSettings.TimeZone = $Timezone
        $Web.Update()
        Invoke-PnPQuery
        Write-host "`Regional settings Updated Successfully!" -ForegroundColor Green
    }
    else
    {
        Write-host "Time Zone $TimezoneName not found!" -ForegroundColor Red
    }
     
    Disconnect-PnPOnline
}
```

### Possible changes on the script

If you want to **exclude** OneDrive Sites from the list of sites to change the region, modify _row 10_ as below:

```powershell
$AllSites = Get-PnPTenantSite
```

If you want to change the region **only** on OneDrive Sites, modify _row 10_ as below:

```powershell
$AllSites = Get-PnPTenantSite -IncludeOneDriveSites -Filter "Url -like '-my.sharepoint.com/personal/'"
```