#Report #SharePointOnline #OneDrive #PowerShell #PnP #SiteCollection #Shortcut 

<br>

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$OneDriveURL = "https://<Domain>-my.sharepoint.com/personal/<UserUPN>"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $PageRelativeUrl,
        $TargetURL
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        RelativeUrl = $PageRelativeUrl
        TargetURL = $TargetURL
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
$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "OneDriveShortCuts"
$FolderName = $ReportName + "_" + $Date
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

try {
    Connect-PnPOnline -Url $OneDriveURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to OneDrive"

    $collListItems = Get-PnPListItem -List "Documents"
    Add-ScriptLog -Color Cyan -Msg "Collected List Items: $($collListItems.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}


$ItemCounter = 0
ForEach($oItem in $collListItems) {

    # Adding notification and logs
    $PercentComplete = [math]::Round($ItemCounter/$collListItems.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Item: $($oItem.FieldValues.FileRef)"
    $ItemCounter++
    
    Try {
        
        if($oItem.FieldValues.A2ODExtendedMetadata){
            Add-ScriptLog -Color Green -Msg "FOUND!"
            $jsonobject= ConvertFrom-Json $oItem.FieldValues.A2ODExtendedMetadata
            Add-ReportRecord -PageRelativeUrl $oItem.FieldValues.FileRef -TargetURL $jsonobject.riwu
            #$oItem.FieldValues | Format-List -Property *
            #Add-ScriptLog -Color Green -Msg $jsonobject.riwu
        }
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($oItem.FieldValues.FileRef)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
    }
}

if($collListItems.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collListItems.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```
