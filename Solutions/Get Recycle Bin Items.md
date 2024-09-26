#Report  #OneDrive #SharePointOnline #RecycleBin #PnP 

<br>

## Recycle bin for a Single Site

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL= "https://Domain.sharepoint.com/sites/SiteName"
$ClientId = "00000000-0000-0000-0000-000000000000"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $Item,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteUrl = $SiteURL
        ItemTitle = $Item.Title
        OriginalLocation = $Item.DirName
        SizeMB = [Math]::Round(($Item.Size/1MB),1)
        Remarks = $Remarks
        })

    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog($Color, $Msg)
{
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm K"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}

# Create logs location
$FolderPath = "$Env:USERPROFILE\Documents\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "SharePointGroupMembersReport"
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
    Connect-PnPOnline -Url $SiteURL -ClientId $ClientId -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to Site"

    $RecycleBinItems = Get-PnPRecycleBinItem
    Add-ScriptLog -Color Cyan -Msg "Collected recycle bin items $($RecycleBinItems.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

Add-ScriptLog -Color Yellow -Msg "Processing recycle bin items"

ForEach($Item in $RecycleBinItems)
{
    try {
        Add-ReportRecord -Item $Item
    }
    catch {
        Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Continue
    }
}
Add-ScriptLog -Color Cyan -Msg "100% Completed - Script finished"
Add-ScriptLog -Color Cyan -Msg "Logs generated at $($LogsOutput)"
```