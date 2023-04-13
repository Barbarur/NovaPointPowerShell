#Automation #OneDrive #SharePointOnline #RecycleBin #PnP

<br>

```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################
$SiteURL = "https://<DOMAIN>-admin.sharepoint.com"
$StartDate = "2022-01-31 12:00:00 AM"
$EndDate = "2022-03-1 12:00:00 AM"
$DeletedBy = "<USER@EMAIL.COM>"



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################
# Add new record on the report
Function Add-ReportRecord($Item, $Action, $Remarks)
{
    $Record = New-Object PSObject -Property ([ordered]@{
        "File Name"             = $Item.Title
        "Deleted by"            = $Item.DeletedByEmail
        "Deleted time"          = $Item.DeletedDate
        "Original Location"     = $Item.DirName
        "Action"                = $Action
        "Remarks"               = $Remarks
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
$FolderPath = "$Env:USERPROFILE\Documents\NovaPoint\Automation\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "RestoreRecycledFiles"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
Function Get-DeletedItems($Batch) {
    Add-ScriptLog -Color Cyan -Msg "Getting deleted files for Batch #$($Batch)"
    return Get-PnPRecycleBinItem -RowLimit 10000 | Where-Object { $_.DeletedDate -gt $StartDate -and $_.DeletedDate -lt $EndDate -and $_.DeletedByEmail -eq $DeletedBy -and $_.id -notin $ErrorItems}
}


# Connect to SharePoint Site and collect subsites
try {
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

# Get All Items Deleted in the Past Days
$ErrorItems = @()
$BatchCounter = 1
$DeletedItems = Get-DeletedItems -Batch $BatchCounter
While($null -ne $DeletedItems) {
    $ItemCounter = 0

    # Restore Recycle bin items matching given query
    ForEach ($Item in $DeletedItems) {
        # Adding notification and logs
        $PercentComplete = [math]::Round($ItemCounter/$DeletedItems.Count*100,2)
        $Msg = "$($PercentComplete) - Adding record $($Item["FileRef"])"
        Add-ScriptLog -Color yellow -Msg "Batch $($BatchCounter) - $($PercentComplete)% Completed - Restoting $($Item.ItemType) '$($Item.Title)' at '$($Item.DirName)'"
        $ItemCounter++
                
        Try {
            #Restore-PnpRecycleBinItem -Identity $Item -Force -ErrorAction Stop
            Add-ReportRecord -Item $Item -Action "Restored"
        }

        Catch {
            If ($_.Exception.Message -like "*with this name*already exists*") {
                Add-ReportRecord -Item $Item -Action "Error" -Remarks "A file with this name already exists on the same location"
                Add-ScriptLog -Color Magenta -Msg "$($Item.ItemType): '$($Item.Title)' already exists in target location. Deleting $($Item.ItemType) from recycle bin"
            }
            Else {
                Add-ReportRecord -Item $Item -Action "Error" -Remarks $_.Exception.Message
                Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
            }
            $ErrorItems += $Item.id
        }
    }
    
    Add-ScriptLog -Color Red -Msg "Error list includes $($ErrorItems.count) items"
    $BatchCounter++
    $DeletedItems = Get-DeletedItems -Batch $BatchCounter
}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$DeletedItems.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"

```