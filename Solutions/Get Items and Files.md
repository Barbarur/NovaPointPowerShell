#Report #SharePointOnline #PowerShell #PnP #DocumentLibrary #ItemList #FileSize #Versioning 

<br>

## Files in a Document Library

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL= "https://<DOMAIN>.sharepoint.com/sites/<SITENAME>"
$LibraryName = "<LIBRARYNAME>"
$RelativePath = "*/MainFolder>/<SubFolder1>/<SubFolder2>/*" # Keep the * ad the beginning and the end


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

    $collItems = Get-PnPListItem -List $LibraryName -PageSize 3000 | Where-Object { $_["FileRef"] -like $RelativePath }
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

## Files and Items in a Site
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
        $List,
        $Item,
        $ItemName,
        $FileSizeBytes,
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
        "Version No" = $Item["_UIVersionString"]
        "Versions Qty" = $Versions.Count
        "ItemSize(MB)" = [Math]::Round(($FileSizeBytes/1MB),1)
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
$ReportName = "ItemReport"
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

    $collLists = Get-PnPList | Where-Object { $_.Hidden -eq $False }
    Add-ScriptLog -Color Cyan -Msg "Collected all Lists: $($collLists.Count)"
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

                Add-ReportRecord -List $oList -Item $oItem -ItemName $oItem["FileLeafRef"] -FileSizeBytes $oItem["File_x0020_Size"]

            }

            elseif ($List.BaseType -eq "GenericList") {

                $Attachments = Get-PnPProperty -ClientObject $oItem -Property AttachmentFiles
                $AttachmentFileSizeTotal = 0
                ForEach ($Attachment in $Attachments) {

                    $AttachmentFile = Get-PnPFile -Url $Attachment.ServerRelativeUrl -AsFileObject
                    $AttachmentFileSizeTotal += $AttachmentFile.Length
                }

                Add-ReportRecord -List $oList -Item $oItem -ItemName $oItem["Title"] -FileSizeBytes $AttachmentFileSizeTotal

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

Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## Files in all libraries of a site

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
        $List,
        $Item,
        $Remarks
    )

    $Versions = Get-PnPProperty -ClientObject $Item -Property Versions

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteURL = $SiteURL
        LibraryName = $List.Title
        ItemID = $Item.Id
        ItemName = $Item["FileLeafRef"]
        ItemType = $Item.FileSystemObjectType
        ItemURL = $Item["FileRef"]
        Version = $Item["_UIVersionString"]
        VersionsCount = $Versions.Count
        ItemSizeMB = [Math]::Round(($Item["File_x0020_Size"]/1MB),1)
        TotalItemSizeMB = [Math]::Round(($Item["SMTotalSize"].LookupId/1MB),1)
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
$ReportName = "ItemReport"
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
    Connect-PnPOnline -Url $SiteURL -Interactive -ClientId $ClientId -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to Site"

    $collLists = Get-PnPList | Where-Object { $_.Hidden -eq $False -and $_.BaseType -eq "DocumentLibrary"}
    Add-ScriptLog -Color Cyan -Msg "Collected all Lists: $($collLists.Count)"
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
        
        $collItems = Get-PnPListItem -List $oList.Title -PageSize 2000 | Where-Object { $_.FileSystemObjectType -ne "Folder"}

        foreach ( $oItem in $collItems) {
            
            $PercentComplete = [math]::Round( $PercentComplete + $ItemCounterStep / $collItems.Count * 100, 2)
            Add-ScriptLog -Color DarkYellow -Msg "$($PercentComplete)% Completed - Processing Item '$($oItem["FileRef"])'"
            
            Add-ReportRecord -List $oList -Item $oItem
        }
    }
    Catch {

        Add-ScriptLog -Color Red -Msg "Error while processing List '$($oList.Title)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -List $oList -Item $oItem -Remarks $_.Exception.Message
    
    }
}

Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## Files and Items in all Sites
```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
# SharePoint Admin Center URL
$AdminSiteURL = "https://<Domain>-admin.sharepoint.com"

# SharePoint Admin Account
$SiteCollAdmin = "<admin@email.com>" 

# Path to the CSV file with the list of Site URLs. File should have a single column with header "Url".
# Leave it empty to go through all Site Collections, this will make the script to run long time and high provability tp crash
$CSVFile = "" 



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $SiteUrl,
        $ListTitle,
        $Item,
        $ItemName,
        $FileSizeBytes,
        $FileSizeTotalBytes,
        $Remarks
    )
    
    $Versions = Get-PnPProperty -ClientObject $Item -Property Versions

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteUrl = $SiteUrl
        ListTitle = $ListTitle
        ID = $Item.Id
        Name = $ItemName
        Type = $Item.FileSystemObjectType
        Extension = $Item["File_x0020_Type"]
        ItemURL = $Item["FileRef"]
        Created = $Item["Created"]
        Createdby = $Item["Author"].Email
        Modified = $Item["Modified"]
        Modifiedby = $Item["Editor"].Email
        VersionNo = $Item["_UIVersionString"]
        VersionsQty = $Versions.Count
        ItemSizeMB = [Math]::Round(($FileSizeBytes/1MB),1)
        TotalItemSizeMB = [Math]::Round(($FileSizeTotalBytes/1MB),1)
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
$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "ItemReport"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Find-Lists {
    param (
        $SiteUrl
    )
    
    Add-ScriptLog -Color White -Msg "Finding Libraries and Lists in Site '$($SiteUrl)'"

    $collLists = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.IsSystemList -ne $True}
    Add-ScriptLog -Color White -Msg "Collected all Lists: $($collLists.Count)"
    
    ForEach($oList in $collLists) {

        Try {
            
            Find-Items -SiteUrl $SiteUrl -List $oList

        }
        Catch {

            Add-ScriptLog -Color Red -Msg "Error while processing List '$($oList.Title)'"
            Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
            Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
            Add-ReportRecord -SiteUrl $SiteUrl -ListTitle $oList.Title -Remarks $_.Exception.Message
        
        }
    }
}

function Find-Items {
    param (
        $SiteUrl,
        $List
    )
    
    Add-ScriptLog -Color White -Msg "Finding '$($List.ItemCount)' Files and Items in '$($List.Title)'"

    $collItems = Get-PnPListItem -List $List.Title -PageSize 5000

    foreach ( $oItem in $collItems) {
        
        Add-ScriptLog -Color White -Msg "Processing '$($oItem["FileRef"])'"

        If($List.BaseType -eq "DocumentLibrary") {

            Add-ReportRecord -SiteUrl $SiteUrl -ListTitle $List.Title -Item $oItem -ItemName $oItem["FileLeafRef"] -FileSizeBytes $oItem["File_x0020_Size"] -FileSizeTotalBytes $FileSizeTotalBytes = $oItem["SMTotalSize"].LookupId

        }

        elseif ($List.BaseType -eq "GenericList") {

            $Attachments = Get-PnPProperty -ClientObject $oItem -Property AttachmentFiles
            $AttachmentFileSizeTotal = 0
            ForEach ($Attachment in $Attachments) {

                $AttachmentFile = Get-PnPFile -Url $Attachment.ServerRelativeUrl -AsFileObject
                $AttachmentFileSizeTotal += $AttachmentFile.Length
            }

            Add-ReportRecord -SiteUrl $SiteUrl -ListTitle $List.Title -Item $oItem -ItemName $oItem["Title"] -FileSizeBytes $AttachmentFileSizeTotal -FileSizeTotalBytes $AttachmentFileSizeTotal

        }
    }
}


try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"
    
    if($CSVFile) {
        $collSiteCollections = Import-CSV $CSVFile
    }
    else {
        $collSiteCollections = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    }
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

        Find-Lists -SiteUrl $oSite.Url
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site '$($oSite.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $oSite.Url -Remarks $_.Exception.Message
    }

    Connect-PnPOnline -Url $oSite.Url -Interactive
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collLists.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```