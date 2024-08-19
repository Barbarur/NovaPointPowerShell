#Report #SharePointOnline #PowerShell #PnP #DocumentLibrary #ItemList #FileSize #Versioning #PHL

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$AdminSiteURL = "https://Domain-admin.sharepoint.com"
$SiteCollAdmin = "admin@email.com" 



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $SiteUrl,
        $Item,
        $VersionsCount,
        $ItemSizeMB,
        $TotalItemSizeMB,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteUrl = $SiteUrl
        Name = $Item["FileLeafRef"]
        ItemURL = $Item["FileRef"]
        Created = $Item["Created"]
        Createdby = $Item["Author"].Email
        Modified = $Item["Modified"]
        Modifiedby = $Item["Editor"].Email
        VersionNo = $Item["_UIVersionString"]
        VersionsQty = $VersionsCount
        ItemSizeMB = $ItemSizeMB
        TotalItemSizeMB = $TotalItemSizeMB
        Remarks = $Remarks
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
$FolderPath = "$Env:USERPROFILE\Documents\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "PHLItemReport"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Get-PHLItems {
    param (
        $SiteUrl
    )
    
    $oList = Get-PnPList -Identity "Preservation Hold Library"
    if ( $null -eq $oList) {
        return
    }

    Add-ScriptLog -Color White -Msg "Processing '$($oList.ItemCount)' Items"

    $collItems = Get-PnPListItem -List $oList.Title -PageSize 5000

    foreach ( $oItem in $collItems) {
        $Versions = Get-PnPProperty -ClientObject $oItem -Property Versions
        Add-ReportRecord -SiteUrl $SiteUrl -Item $oItem -VersionsCount $Versions.Count -ItemSizeMB ([Math]::Round(($oItem["File_x0020_Size"]/1MB),1)) -TotalItemSizeMB ([Math]::Round(($oItem["SMTotalSize"].LookupId/1MB),1))
    }
}


try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"
    
    $collSiteCollections = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: $($collSiteCollections.count)"    
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSite.URL)"
    $ItemCounter++

    Try {
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin -ErrorAction Stop

        Connect-PnPOnline -Url $oSite.Url -Interactive

        Get-PHLItems -SiteUrl $oSite.Url

        $collSubSites = Get-PnPSubWeb -Recurse

        ForEach($oSubsite in $collSubSites) {
            try {
                Get-PHLItems -SiteUrl $oSubsite.Url
            }
            catch {
                Add-ScriptLog -Color Red -Msg "Error while processing Subsite '$($oSubsite.Url)"
                Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
                Add-ScriptLog -Color Red -Msg "Error trace: '$($_.InvocationInfo.ScriptLineNumber)'"
            }
        }
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site '$($oSite.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
    }

    Connect-PnPOnline -Url $oSite.Url -Interactive
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}

Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```