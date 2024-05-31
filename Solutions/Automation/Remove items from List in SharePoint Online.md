#Automation #PnP #PowerShell #PreservationHolLibrary #PHL #RetentionPolicy #SharePointOnline #Remove #File #ListItem

<br>

## Remove Files from a Folder

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL = "https://Domain.sharepoint.com/sites/SitName"
$ListName ="Documents"
$FolderServerRelativeUrl = "/sites/SiteName/Library/Samples/Demo"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################
# Add new record on the report
Function Add-ReportRecord{
    param (
        $Item,
        $Remarks
    )
    $Record = New-Object PSObject -Property ([ordered]@{
        FileTitle = $Item.FileTitle
        FilePath = $Item.FilePath
        Remarks = $Remarks
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog {
    param (
        $Color,
        $Msg
    )

    Write-host -f $Color $Msg
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
}

# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "RemoveFiles"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

$ItemsColl = @()
try {
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to Site and Collected Items"

    $FolderFilter = $FolderServerRelativeUrl + "*"
    try {
        $ItemsList = Get-PnPListItem -List $ListName -FolderServerRelativeUrl $FolderServerRelativeUrl | Where-Object { $_["FileRef"] -Like $FolderFilter }
        Add-ScriptLog -Color Cyan -Msg "Collected Items: $($ItemsList.Count)"
    }
    catch {
        $list = Get-PnPList -Identity $ListName

        Add-ScriptLog -Color Yellow -Msg "Folder containes more than 5k items. Script will have to collect all items in the list and then filter based on the path."
        Add-ScriptLog -Color Yellow -Msg "Library has $($list.ItemCount) items"

        $ItemsList = Get-PnPListItem -List $ListName -PageSize 5000 -ErrorAction Stop | Where-Object { $_["FileRef"] -Like $FolderFilter }
        Add-ScriptLog -Color Cyan -Msg "Collected Items: $($ItemsList.Count)"
    }

    ForEach ($Item in $ItemsList)
    {
        $ItemsColl += New-Object PSObject -Property ([ordered]@{
            FileID = $Item.Id
            FileTitle = $Item["FileLeafRef"]
            FilePath = $Item["FileRef"]
            Created = $Item["Created"]
            Createdby = $Item["Author"].Email
            Modified = $Item["Modified"]
            Modifiedby = $Item["Editor"].Email
            })
    }

    $ItemsColl = $ItemsColl | Sort-Object -Descending -Property FilePath

}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
ForEach ($Item in $ItemsColl)
{
        $PercentComplete = [math]::Round($ItemCounter/$ItemsColl.Count * 100, 1)
        Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Removing item: $($Item.FilePath)"
        $ItemCounter++

    try {
        # Remove-PnPListItem -List $ListName -Identity $Item.FileID -Recycle -Force -ErrorAction Stop
        Add-ReportRecord -Item $Item -Remarks "Completed" 
    }
    catch {
        Add-ScriptLog -Color Red -Msg "Error while processing $($Item.FilePath)"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.Exception.ScriptLineNumber)'"
        Add-ReportRecord -Item $Item -Remarks $_.Exception.Message
    }
}

Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```
## Remove files from Document Library

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL = "https://<Domain>.sharepoint.com/sites/<SitName>"
$ListName ="Documents"
$CreatedAfter = "2023-11-26 12:00:00 AM"
$CreatedBefore = "2023-11-30 12:00:00 AM"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################
# Add new record on the report
Function Add-ReportRecord{
    param (
        $Item,
        $Remarks
    )
    $Record = New-Object PSObject -Property ([ordered]@{
        FileTitle = $Item["FileLeafRef"]
        FilePath = $Item["FileRef"]
        Created = $Item["Created"]
        Createdby = $Item["Author"].Email
        Modified = $Item["Modified"]
        Modifiedby = $Item["Editor"].Email
        Remarks = $Remarks
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog {
    param (
        $Color,
        $Msg
    )

    Write-host -f $Color $Msg
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
}

# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "RemoveFiles"
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
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to Site and Collected Items"

    $ItemsList = Get-PnPListItem -List $ListName -PageSize 3000 -ErrorAction Stop| Where-Object { $_["Created"] -gt $CreatedAfter -and $_["Created"] -lt $CreatedBefore }
    Add-ScriptLog -Color Cyan -Msg "Collected Items: $($ItemsList.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
ForEach ($Item in $ItemsList)
{
        $PercentComplete = [math]::Round($ItemCounter/$ItemsList.Count * 100, 1)
        Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Removing item: $($Item["FileLeafRef"])"
        $ItemCounter++

    try {
        # Remove-PnPListItem -List $ListName -Identity $Item.Id -Recycle -Force -ErrorAction Stop
        Add-ReportRecord -Item $Item -Remarks "Completed" 
    }
    catch {
        Add-ScriptLog -Color Red -Msg "Error while processing $($Item["FileLeafRef"])"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.Exception.ScriptLineNumber)'"
        Add-ReportRecord -Item $Item -Remarks $_.Exception.Message
    }
}

if($ItemsList.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$ItemsList.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## Remove files from Preservation Hold Library

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
