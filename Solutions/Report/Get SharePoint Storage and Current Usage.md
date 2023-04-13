#Report #PnP #PowerShell #SharePointOnline #Storage #SiteCollection 

<br>
SharePoint Online Storage Usage report [can take up to 48h to update](https://learn.microsoft.com/en-us/sharepoint/manage-site-collection-storage-limits). In some occasions when a quicker report is needed, this script can help to deliver this information in a matter of minutes.

<br>

```powershell
# Define Parameters 
$AdminSiteURL= "https://DOMAIN-admin.sharepoint.com" 
$ItemCounter = 0  
$StorageUsed = 0 

Connect-PnPOnline -Url $AdminSiteURL –Interactive 

$Tenant = Get-PnPTenant 

# Get all Sites and itinerate 
$ListSites = Get-PnPTenantSite | Where{ ($_.Title -notlike "") } 
ForEach($Site in $ListSites) 
{ 

    #Status notification 
    $ItemCounter++ 
    $PercentComplete = [math]::Round($ItemCounter/$ListSites.Count*100,1) 
    Write-Progress -PercentComplete $PercentComplete -Activity "Completed $($PercentComplete)%" -Status "Site '$($Site.URL)" 

    $StorageUsed = $StorageUsed + $Site.StorageUsageCurrent 

} 

# Close status notification 
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)" 

$TotalStorageTb = ('{0:N2}' -f ([math]::Round($Tenant.StorageQuota/1024/1024,2))) 

$StorageUsedTb = ('{0:N2}' -f ([math]::Round($StorageUsed/1024/1024,2))) 

$StorageAvailable = $Tenant.StorageQuota - $StorageUsed 
$StorageAvailableTb = ('{0:N2}' -f ([math]::Round($StorageAvailable/1024/1024,2))) 


Write-Host -f Yellow "Total Storage:"$TotalStorageTb 
Write-Host -f Yellow "Current Consumption:"$StorageUsedTb 
Write-Host -f Yellow "Storge available:"$StorageAvailableTb
```