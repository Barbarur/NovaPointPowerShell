#PnP #PowerShell #SiteCollection #SiteAdmin #SPOService

<br>

## Option 1: Using SharePointOnlinePowerShell

### Add user as Site Admin

```powershell
# Define Parameters
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com"
$SiteCollAdmin = "<USER@EMAIL.com>"

# Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL

# Get all Site collections
$collSiteCollections = Get-SPOSite -Limit ALL -IncludePersonalSite $False | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
Write-Host 'Total number of Site Collections:'$SitesList.Count

$ItemCounter = 0
# Itinerate across all SitesList
ForEach($Site in $collSiteCollections){
    # Status notification
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count*100,1)
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
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com"
$SiteCollAdmin = "<USER@EMAIL.com>"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord($SiteURL, $Action)
{
    $Record = New-Object PSObject -Property ([ordered]@{
        "Site URL"          = $SiteURL
        "Action"            = $Action
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog($Color, $Msg)
{
    Write-host -f $Color $Msg
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
}

# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\NovaPoint\QuickFix\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "IDMismatch"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
try {
    Connect-SPOService -Url $AdminSiteURL -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-SPOSite -Limit ALL -IncludePersonalSite $False | Where-Object { ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: Total: $($collSiteCollections.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSite.Title)"
    $ItemCounter++

    Try {

        Set-SPOUser -Site $oSite.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $False
        Add-ReportRecord -SiteURL $oSite.Url -Action "Removed user as Site Collection Admin"
        
    }
    Catch {

        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($Site.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        Add-ReportRecord -SiteURL $oSite.Url -Action $_.Exception.Message
        
    }    
}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

### Possible changes on the script

If you want to add the new Admin **also** in all OneDrive Sites:

```powershell
$collSiteCollections = Get-SPOSite -Limit ALL -IncludePersonalSite $True | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
```

If you want to add the new Admin **only** OneDrive Sites:

```powershell
$collSiteCollections = Get-SPOSite -Template "SPSPERS" -Limit ALL -IncludePersonalSite $True | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
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