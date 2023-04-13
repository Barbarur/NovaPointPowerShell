#Automation #PnP #PowerShell #SharePointOnline #SiteCollection #Subsite

<br>

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL = "https://DOMAIN-admin.sharepoint.com"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################
# Add new record on the report
Function Add-ReportRecord($Status, $Remarks) {
    $Record = New-Object PSObject -Property ([ordered]@{
        "Site URL"          = $Site.URL
        "Status"            = $Status
        "Remarks"           = $Remarks
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

# Add Log of the Script
Function Add-ScriptLog($Color, $Msg) {
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm K"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}

# Create Report location
$ReportName = "FolderPathReport"
$Date = Get-Date -Format FileDateTime
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

# Connect to SharePoint Site and collect subsites
try {
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to Site Collection"

    $SubSites = Get-PnPSubWeb -Recurse -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Collected Subsites"
    Add-ScriptLog -Color Cyan -Msg "Number of Subsites: $($SubSites.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
}

# Collect iterate through Subsites
$ItemCounter = 0
ForEach($Site in $SubSites)
{
    # Adding notification and logs
    $PercentComplete = [math]::Round($ItemCounter/$SubSites.Count * 100, 0)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Deleting subsite '$($Site.URL)'"

    try {
        Remove-PnPWeb -Identity $Site.id -Force
        Add-ReportRecord -Status "Deleted"

    }
    catch {
        Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
        Add-ReportRecord -Status "Error" -Remarks $_.Exception.Message
    }

    $ItemCounter++
}
# Close status notification
$PercentComplete = [math]::Round($ItemCounter/$SubSites.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Script finished running"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```