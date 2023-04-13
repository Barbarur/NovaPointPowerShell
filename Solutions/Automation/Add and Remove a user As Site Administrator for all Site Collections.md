#PnP #PowerShell #SiteCollection #SiteAdmin #SPOService

<br>

## Option 1: Using SharePointOnlinePowerShell

### Add user as Site Admin

```powershell
# Define Parameters
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com"
$SiteCollAdmin = "<USER@EMAIL.com>"

# Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL -Interactive

# Get all Site collections
$SitesList = Get-SPOSite -Limit ALL -IncludePersonalSite $False | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
Write-Host 'Total number of Site Collections:'$SitesList.Count

$ItemCounter = 0
# Itinerate across all SitesList
ForEach($Site in $SitesList){
    # Status notification
    $PercentComplete = [math]::Round($ItemCounter/$SitesList.Count*100,1)
    Write-Progress -PercentComplete $PercentComplete -Activity "Processing $($PercentComplete)%" -Status "Site $($Site.URL)"
    $ItemCounter++

    Try{
        Set-SPOUser -Site $Site.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $True
        Write-host -f Green "Added Site Collection Administrator to $($Site.URL)"
        }
    Catch{
        Write-Host -f Red $Site.url"ERROR!"
        }
    }
# Close status notification
Write-Progress -Activity "Processing $($PercentComplete)%" -Status "Site '$($Site.URL)"

Write-Host -b Green "Completed Running Script"
```

<br>

### Remove user as Site Admin

```powershell
# Define Parameters
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com"
$SiteCollAdmin = "<USER@EMAIL.com>"

# Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL

# Get all Site collections
$SitesList = Get-SPOSite -Limit ALL -IncludePersonalSite $False | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
Write-Host 'Total number of Site Collections:'$SitesList.Count

$ItemCounter = 0
# Itinerate across all SitesList
ForEach($Site in $SitesList){
    # Status notification
    $PercentComplete = [math]::Round($ItemCounter/$SitesList.Count*100,1)
    Write-Progress -PercentComplete $PercentComplete -Activity "Processing $($PercentComplete)%" -Status "Site $($Site.URL)"
    $ItemCounter++

    Try{
        Set-SPOUser -Site $Site.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $False
        Write-host -f Green "Removed Site Collection Administrator from $($Site.URL)"
        }
    Catch{
        Write-Host -f Red $Site.url"ERROR!"
        }
    }
# Close status notification
Write-Progress -Activity "Processing $($PercentComplete)%" -Status "Site '$($Site.URL)"

Write-Host -b Green "Completed Running Script"
```

<br>

### Possible changes on the script

If you want to add the new Admin **also** in all OneDrive Sites:

```powershell
$SitesList = Get-SPOSite -Limit ALL -IncludePersonalSite $True | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
```

If you want to add the new Admin **only** OneDrive Sites:

```powershell
$SitesList = Get-SPOSite -Template "SPSPERS" -Limit ALL -IncludePersonalSite $True | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
```

<br>

<br>

## Option #2: Using PnP

### Add user as Site Admin

```powershell
# Define Parameters
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com"
$SiteCollAdmin = "<USER@EMAIL.com>"

# Connect to SharePoint Online Admin Center
Connect-PnPOnline -Url $AdminSiteURL
 
# Get all Site collections
$SitesList = Get-PnPTenantSite | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
Write-Host 'Total number of Site Collections:'$SitesList.Count

$ItemCounter = 0
# Itinerate across all SitesList
ForEach($Site in $SitesList){
    # Status notification
    $PercentComplete = [math]::Round($ItemCounter/$SitesListList.Count*100,1)
    Write-Progress -PercentComplete $PercentComplete -Activity "Processing $($PercentComplete)%" -Status "Site $($Site.URL)"
    $ItemCounter++

    Try{
        Set-PnPTenantSite -Url $Site.Url -Owners $SiteCollAdmin
        Write-host -f Green "Added Site Collection Administrator to $($Site.URL)"
    }
    Catch{
        Write-Host -f Red $Site.url"ERROR!"
    }
}
# Close status notification
Write-Progress -Activity "Processing $($PercentComplete)%" -Status "Site '$($Site.URL)"

Write-Host -b Green "Completed Running Script"
```

<br>

### Remove user as Site Admin

```powershell
# Define Parameters
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com"
$SiteCollAdmin = "<USER@EMAIL.com>"

# Connect to SharePoint Online Admin Center
Connect-PnPOnline -Url $AdminSiteURL -Interactive

# Get all Site collections
$SitesList = Get-PnPTenantSite | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
Write-Host 'Total number of Site Collections:'$SitesList.Count

# Itinerate across all SitesList
$ItemCounter = 0
ForEach($Site in $SitesList){
    # Status notification
    $PercentComplete = [math]::Round($ItemCounter/$SitesListList.Count*100,1)
    Write-Progress -PercentComplete $PercentComplete -Activity "Processing $($PercentComplete)%" -Status "Site $($Site.URL)"
    $ItemCounter++

    Try{
        Connect-PnPOnline $Site.URL -Interactive
        Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
        Write-host -f Green "Removed Site Collection Administrator from $($Site.URL)"
    }
    Catch{
        Write-Host -f Red $Site.url"ERROR!"
    }
}
# Close status notification
Write-Progress -Activity "Processing $($PercentComplete)%" -Status "Site '$($Site.URL)"

Write-Host -b Green "Completed Running Script"
```

<br>

### Possible changes on the script

If you want to add the new Admin **also** in all OneDrive Sites:

```powershell
$SitesList = Get-PnPTenantSite -IncludeOneDriveSites | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
```

If you want to add the new Admin **only** OneDrive Sites:

```powershell
$SitesList = Get-PnPTenantSite -IncludeOneDriveSites -Filter "Url -like '-my.sharepoint.com/personal/'" | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
```