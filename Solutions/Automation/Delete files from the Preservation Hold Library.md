#Automation #PnP #PowerShell #PreservationHolLibrary #PHL #RetentionPolicy #SharePointOnline 

<br>

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL = "https://<DOMAIN>.sharepoint.com/sites/<SITENAME>"
$ListName ="Preservation Hold Library"
$FileName = "*XXX*"
$User = "*<USER@RMAIL.COM>*"
$StartDate = "2022-01-31 12:00:00 AM"
$EndDate = "2022-03-1 12:00:00 AM"
$FileSizeMaxMB = 123.45 # Size in Mb
$FolderPath = "$Env:USERPROFILE\Documents\"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################
# Add new record on the report
Function Add-ReportRecord($Item)
{
    $Record = New-Object PSObject -Property ([ordered]@{
        "File Name"                = $Item.FieldValues.FileLeafRef
        "File Original Path"       = $Item.FieldValues.PreservationOriginalURL
        "Modified"                 = $Item.FieldValues.Modified
        "Modified by"              = $Item.FieldValues.Modified_x0020_By
        "Date Preserved"           = $Item.FieldValues.PreservationDatePreserved
        "Action"                   = $Action
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
$ReportName = "DeletedFilesPHLReport"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"
$ReportOutput = $FolderPath + $FolderName + "\" + $ReportName + ".csv"

# Create logs file
$LogsName = $ReportName + "_Logs.txt"
$LogsOutput = $FolderPath + $FolderName + "\" + $LogsName

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
try {
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    $FileSizeMax  = $FileSizeMaxMB * 1024 * 1024
    $ItemsList = Get-PnPListItem -List $ListName -PageSize 3000 -ErrorAction Stop| Where-Object { $_.FieldValues.File_x0020_Size -lt $FileSizeMax -and $_.FieldValues.Modified_x0020_By -like $User -and $_.FieldValues.PreservationDatePreserved -gt $StartDate -and $_.FieldValues.PreservationDatePreserved -lt $EndDate -and $_.FieldValues.FileLeafRef -like $FileName}
    Add-ScriptLog -Color Cyan -Msg "Connected to Site and Collected Items"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
}

# Iterate through the items
$ItemCounter = 0
ForEach ($Item in $ItemsList)
{
    # Adding notification and logs
        $PercentComplete = [math]::Round($ItemCounter/$ItemsList.Count * 100, 1)
        Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Removing item: $($Item.FieldValues.FileRef)"
        $ItemCounter++

    try {
        Remove-PnPListItem -List $ListName -Identity $Item.Id -Recycle -Force -ErrorAction Stop
        Add-ReportRecord -Item $Item -Action "Completed" 
    }
    catch {
        Add-ScriptLog -Color Red -Msg "ERROR! Removing item: $($Item.FieldValues.FileRef)"
        Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
        Add-ReportRecord -Item $Item -Action "Error"
    }
}
# Close status notification
$PercentComplete = [math]::Round($ItemCounter/$ItemsList.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```