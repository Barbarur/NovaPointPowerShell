#Report  #OneDrive #SharePointOnline #RecycleBin #PnP 

<br>

## Single Site

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL= "https://Domain.sharepoint.com/sites/SiteName"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################
function Add-ReportRecord {
    param (
        $SiteURL,
        $Item
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        "Site URL" = $SiteURL
        "Item Title" = $Item.Title
        "Item Type" = $Item.ItemType
        "Item State" = $Item.ItemState
        "Date Deleted" = $Item.DeletedDate
        "Deleted By" = $Item.DeletedByEmail
        "Original Location" = $Item.DirName
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append

}

function Add-ScriptLog($Color, $Msg) {
    Write-host -f $Color $Msg
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
}

$FolderPath = "$Env:USERPROFILE\Documents\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "RecycleBinItems"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Logs will be generated at $($LogsOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
function Get-RecycleBinItems {

    $RecycleBinItems = Get-PnPRecycleBinItem
    $ItemCounter = 0
    ForEach ($Item in $RecycleBinItems) {
        $PercentComplete = [math]::Round($ItemCounter/$RecycleBinItems.Count * 100, 2)
        Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed"
        $ItemCounter++

        try {
            Add-ReportRecord -SiteURL $SiteURL -Item $Item
        }
        catch {
            Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
            Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        }

    }
}

try {
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to Site '$($SiteURL)'"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

Get-RecycleBinItems

Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```