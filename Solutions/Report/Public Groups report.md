

<br>

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord {
    param (
        $Group
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        Case = $Group.Name
        HoldPolicy = $Group.SharePointSiteUrl
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog($Color, $Msg) {
    Write-host -f $Color $Msg
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
}

# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "PublicGroups"
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

    Connect-ExchangeOnline -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to EXO"

    $collGroups = Get-UnifiedGroup | Where-Object { $_.AccessType -eq "Public" }
    Add-ScriptLog -Color Cyan -Msg "Groups ollected: $($collGroups.Count)"

}
catch {

    Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
    Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
    Disconnect-ExchangeOnline
    break

}

$ItemCounter = 0
foreach($oGroup in $collGroups) {

    Add-ReportRecord -Group $oGroup
    $ItemCounter++

}
if($collGroups.Count -ne 0) {

    $PercentComplete = [math]::Round( $ItemCounter/$collGroups.Count * 100, 2 )
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"

}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
Disconnect-ExchangeOnline
```
