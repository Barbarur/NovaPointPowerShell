#Report #SharePointOnline #PowerShell #PnP #DocumentLibrary #ItemList #FileSize #Versioning 

<br>

## All Files in Document Library

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL= "https://<DOMAIN>.sharepoint.com/sites/<SITENAME>"
$LibraryName = "<LIBRARYNAME>"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $Item,
        $Remarks
    )
    
    $Versions = Get-PnPProperty -ClientObject $Item -Property Versions

    $Record = New-Object PSObject -Property ([ordered]@{
        "SiteURL" = $SiteURL
        "LibraryName" = $LibraryName
        "Item ID" = $Item.Id
        "Item Name" = $Item["FileLeafRef"]
        "ItemType" = $Item.FileSystemObjectType
        "Item URL" = $Item["FileRef"]
        Created = $Item["Created"]
        "Created by" = $Item["Author"].Email
        Modified = $Item["Modified"]
        "Modified by" = $Item["Editor"].Email
        "Version No" = $Item["_UIVersionString"]
        "Versions Qty" = $Versions.Count
        "ItemSize(MB)" = [Math]::Round(($Item["File_x0020_Size"]/1MB),1)
        "TotalItemSize(KB)" = [Math]::Round(($Item["SMTotalSize"].LookupId/1KB),1)
        "TotalItemSize(MB)" = [Math]::Round(($Item["SMTotalSize"].LookupId/1MB),1)
        "TotalItemSize(GB)" = [Math]::Round(($Item["SMTotalSize"].LookupId/1GB),1)
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
$FolderPath = "$Env:USERPROFILE\Documents\SPOScripts\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "LibraryItemReport"
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
    Add-ScriptLog -Color Cyan -Msg "Connected to Site"

    $collItems = Get-PnPListItem -List $LibraryName -PageSize 3000
    Add-ScriptLog -Color Cyan -Msg "Collected all Items"
    Add-ScriptLog -Color Cyan -Msg "Items Total: $($collItems.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}


$ItemCounter = 0
ForEach($oItem in $collItems) {

    $PercentComplete = [math]::Round($ItemCounter/$collItems.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Item '$($oItem["FileRef"])'"
    $ItemCounter++

    Try {
        
        Add-ReportRecord -Item $oItem

    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Item '$($oItem["FileRef"])"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        Add-ReportRecord -Item $oItem -Remarks $_.Exception.Message
    }
}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## All Files and Items in a Site
```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL= "https://<DOMAIN>.sharepoint.com/sites/<SITENAME>"
$SiteURL= "https://m365x88421522.sharepoint.com/sites/12345678"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $List,
        $Item,
        $ItemName,
        $FileSizeBytes,
        $FileSizeTotalBytes,
        $Remarks
    )
    
    $Versions = Get-PnPProperty -ClientObject $Item -Property Versions

    $Record = New-Object PSObject -Property ([ordered]@{
        "SiteURL" = $SiteURL
        "LibraryName" = $LibraryName
        "Item ID" = $Item.Id
        "Item Name" = $ItemName
        "ItemType" = $Item.FileSystemObjectType
        "Item URL" = $Item["FileRef"]
        Created = $Item["Created"]
        "Created by" = $Item["Author"].Email
        Modified = $Item["Modified"]
        "Modified by" = $Item["Editor"].Email
        "Version No" = $Item["_UIVersionString"]
        "Versions Qty" = $Versions.Count
        "ItemSize(MB)" = [Math]::Round(($FileSizeBytes/1MB),1)
        "TotalItemSize(KB)" = [Math]::Round(($FileSizeTotalBytes/1KB),1)
        "TotalItemSize(MB)" = [Math]::Round(($FileSizeTotalBytes/1MB),1)
        "TotalItemSize(GB)" = [Math]::Round(($FileSizeTotalBytes/1GB),1)
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
$FolderPath = "$Env:USERPROFILE\Documents\SPOScripts\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "LibraryItemReport"
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
    Add-ScriptLog -Color Cyan -Msg "Connected to Site"

    $ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
    $collLists = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists}
    Add-ScriptLog -Color Cyan -Msg "Collected all Lists"
    Add-ScriptLog -Color Cyan -Msg "Lists Total: $($collLists.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}


$ItemCounter = 0
$ItemCounterStep = 1 / $collLists.Count
ForEach($oList in $collLists) {

    $PercentComplete = [math]::Round($ItemCounter/$collLists.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing List '$($oList.Title)'"
    $ItemCounter++

    Try {
        
        $collItems = Get-PnPListItem -List $oList.Title -PageSize 2000

        foreach ( $oItem in $collItems) {
            
            $PercentComplete = [math]::Round( $PercentComplete + $ItemCounterStep / $collItems.Count * 100, 2)
            Add-ScriptLog -Color DarkYellow -Msg "$($PercentComplete)% Completed - Processing Item '$($oItem["FileRef"])'"
            
            If($List.BaseType -eq "DocumentLibrary") {

                Add-ReportRecord -List $oList -Item $oItem -ItemName $oItem["FileLeafRef"] -FileSizeBytes $oItem["File_x0020_Size"] -FileSizeTotalBytes $FileSizeTotalBytes = $oItem["SMTotalSize"].LookupId

            }

            elseif ($List.BaseType -eq "GenericList") {

                $Attachments = Get-PnPProperty -ClientObject $oItem -Property AttachmentFiles
                $AttachmentFileSizeTotal = 0
                ForEach ($Attachment in $Attachments) {

                    $AttachmentFile = Get-PnPFile -Url $Attachment.ServerRelativeUrl -AsFileObject
                    $AttachmentFileSizeTotal += $AttachmentFile.Length
                }

                Add-ReportRecord -List $oList -Item $oItem -ItemName $oItem["Title"] -FileSizeBytes $AttachmentFileSizeTotal -FileSizeTotalBytes $AttachmentFileSizeTotal

            }
        }
    }
    Catch {

        Add-ScriptLog -Color Red -Msg "Error while processing List '$($oList.Title)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        Add-ReportRecord -List $oList -Item $oItem -Remarks $_.Exception.Message
    
    }
}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collLists.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```