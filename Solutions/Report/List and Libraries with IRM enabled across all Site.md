#Report #PowerShell #PnP #ItemList #DocumentLibrary #SiteCollection #Subsite 

<br>

## Run Script

```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com"
$SiteCollAdmin = "<USER@EMAIL.com>"



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################
# Add new record on the report
Function Add-ReportRecord($SiteURL, $LibraryName, $Remarks)
{
    $Record = New-Object PSObject -Property ([ordered]@{
        "Site URL"          = $SiteURL
        "Library Name"      = $LibraryName
        "Remarks"           = $Remarks
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

# Add Log of the Script
Function Add-ScriptLog($Color, $Msg)
{
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}

# Create Report location
$Date = Get-Date -Format "yyyyMMdd_HHmmss"
$ReportName = "IRMLibraryReport"
$FolderName = $Date + "_" + $ReportName
$FolderPath = "$Env:USERPROFILE\Documents\"
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"
$ReportOutput = $FolderPath + $FolderName + "\" + $ReportName + ".csv"

# Create logs file
$LogsName = $ReportName + "_Logs.txt"
$LogsOutput = $FolderPath + $FolderName + "\" + $LogsName

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
Function Find-IRMLibray($SiteURL)
{
    Connect-PnPOnline -Url $SiteURL -Interactive

    $ExcludedListsList = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
    $ListsList = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedListsList}

    ForEach($List in $ListsList) {
        If($list.IrmEnabled -eq $true) {
            Add-ScriptLog -Color Magenta -Msg "$($PercentComplete)% Completed - Checking Site: $($Site.Title) - IRM Enabled for $($List.BaseType) '$($List.Title)'"
            Add-ReportRecord -SiteURL $SiteURL -LibraryName $List.Title
        }
        Else {
            Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Checking Site: $($Site.Title) - IRM NOT enabled for $($List.BaseType) '$($List.Title)'"
        }
    }
}

# Connect to SharePoint Site and collect subsites
try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"
    $SitesList = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections"
    Add-ScriptLog -Color Cyan -Msg "Number of SiteCollections: $($SitesList.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
}

# Itinerate across all SitesList
$ItemCounter = 0
ForEach($Site in $SitesList){
    # Adding notification and logs
    $PercentComplete = [math]::Round($ItemCounter/$SitesList.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Checking Site: $($Site.Title)"
    $ItemCounter++

    Try{
        Set-PnPTenantSite -Url $Site.Url -Owners $SiteCollAdmin -ErrorAction Stop
    }
    Catch{
        Add-ScriptLog -Color Red -Msg "Error adding user as Site Collection Administrator for Site Collection '$($Site.Title)'"
        Add-ReportRecord -SiteURL $Site.Url -Remarks $_.Exception.Message
    }

    Find-IRMLibray -SiteURL $Site.Url

    $SubSites = Get-PnPSubWeb -Recurse

    ForEach($Site in $SubSites) {
        Find-IRMLibray -SiteURL $Site.Url
    }

    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}
# Close status notification
$PercentComplete = [math]::Round($ItemCounter/$SitesList.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```