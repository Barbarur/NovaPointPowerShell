#

<br>

## Delete versions of a File using PnP

```powershell
#Define Parameters
$SiteURL = "https://{DOMAIN}.sharepoint.com/sites/{SITENAME}/"
$FileRelativePath = "/sites/{SITENAME}/Documents/DesiredFolder/File.docx"
$VersionsToKeep = 0

#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin
 
#Get the Context
$Ctx= Get-PnPContext

#Get File Versions
$File = Get-PnPFile -Url $FileRelativePath
$Versions = $File.Versions
$Ctx.Load($File)
$Ctx.Load($Versions)
$Ctx.ExecuteQuery()

#Notification of file collected
Write-host -f Yellow "Scanning File:"$File.Name
$VersionsCount = $Versions.Count
$VersionsToDelete = $VersionsCount - $VersionsToKeep
write-host -f Cyan "`t Total Number of Versions of the File:" $VersionsCount
write-host -f Cyan "`t Total Number of Versions to be deleted:" $VersionsToDelete

If($VersionsToDelete -gt 0)
{
    #Delete versions
    For($i=0; $i -lt $VersionsToDelete; $i++)
    {
        write-host -f Cyan "`t Deleting Version:" $Versions[0].VersionLabel
        $Versions[0].DeleteObject()
     }
     $Ctx.ExecuteQuery()
     Write-Host -f Green "`t Version History is cleaned for the File:"$File.Name
}
```

<br>

## Delete versions of the Files in a folder using PnP

```powershell
# Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME"
$ListName = "LIBRARY NAME"
$FolderServerRelativeURL="/sites/SITENAME/LIBRARYNAME/DesiredFolder/"
$VersionsToKeep = 2
$ReportOutput = "$Env:USERPROFILE\Desktop\VersionsDeletedReport.csv"

Connect-PnPOnline -Url $SiteUTL -Interactive

Function Add-Report(){
    $Global:Results += New-Object PSObject -property ([ordered]@{
        "Item ID"                   = $ItemID
        "Item Name"                 = $ItemName
        "Item URL"                  = $ItemRelativeURL
        "Version No"                = $Version
        "Versions Qty"              = $VersionsqQty
        "Deleted Versions Qty"      = $VersionsToDelete
        "Total Deleted size (MB)"   = $VersionDeletedSizeMB
        })
    }


Function Remove-Versions(){
 
    For($i=0; $i -lt $VersionsToDelete; $i++)
    {

        Write-Host -f Cyan "Removing verion:"$ListVersions[$i].VersionLabel
        Remove-PnPFileVersion -Url $Item["FileRef"] -Identity $ListVersions[$i].id -Force

    }

}


# Get All Items from the List and iterate
$ListItems = Get-PnPListItem -List $ListName -PageSize 3000 -FolderServerRelativeUrl $FolderServerRelativeURL
$ItemCounter = 0
$Global:Results = @()    
ForEach($Item in $ListItems)
    {
    # Report Item infomation
    $ItemID              = $Item.Id
    $ItemName            = $Item["FileLeafRef"]
    $ItemRelativeURL     = $Item["FileRef"]
    $Version             = $Item["_UIVersionString"]
        
    # Process notification start 
    $ItemCounter++
    $PercentComplete = [math]::Round($ItemCounter/$ListItems.Count*100,1)
    Write-Progress -PercentComplete $PercentComplete -Activity "$Msg $PercentComplete%" -Status $ItemRelativeURL

    If($Item.FileSystemObjectType -eq "Folder") {Continue}

    $ListVersions = Get-PnPFileVersion -Url $Item["FileRef"]
    $VersionsqQty = $ListVersions.Count
    $VersionsToDelete = $VersionsqQty - $VersionsToKeep
            
    If($VersionsToDelete -le 0)
    {
        $VersionsToDelete = 0
        $VersionDeletedSizeMB = 0
        Add-Report

    }
    Else
    {

        $FileSizeTotalMB = [Math]::Round(($Item["SMTotalSize"].LookupId/1MB),1)
        $VersionDeletedSizeMB = ($FileSizeTotalMB / $VersionsqQty) / $VersionsToDelete

        Remove-Versions
        Add-Report

    }
            

}
    
# Process notification finish
Write-Progress -Activity "Processing $($Global:ItemCounter)%" -Status $Msg -Completed

# Export the results to CSV
If($Global:Results.count -eq 0){ Write-host -b Red "Report is empty!" }

Else
{

    If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
    $Global:Results | Export-Csv -Path $ReportOutput -NoTypeInformation
    Write-host -b Green "Report Generated Successfully!"
    Write-host -f Green $ReportOutput
}
```

<br>

## Delete versions of all Files in a Document Libraries of a Site

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL= "https://<Domain>.sharepoint.com/sites/<SiteName>"
$VersionsToKeep = 50
$LibraryName = "Documents"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord {
    param (
        $List,
        $Item,
        $VersionsCount,
        $DeletedVersionsCount,
        $VersionDeletedSizeMB
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        ListName = $List.Title
        ListURL = $List.DefaultViewUrl -Replace ('Forms/AllItems.aspx','') -Replace ('AllItems.aspx','')
        ListType = $List.BaseType
        ItemID = $Item.Id
        ItemURL = $Item["FileRef"]
        VersionNo = $Item["_UIVersionString"]
        VersionsHistoryCount = $VersionsCount
        DeletedVersionsCount = $DeletedVersionsCount
        TotalDeletedSizeMB = $VersionDeletedSizeMB
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
$ReportName = "VersionsDeleted"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
Function Remove-Versions {
    param (
        $Item,
        $Versions
    )
 
    For($i=0; $i -lt $VersionsToDelete; $i++)
    {
        Add-ScriptLog -Color Magenta -Msg "Removing verion: $($Versions[$i].VersionLabel)"
        Remove-PnPFileVersion -Url $Item["FileRef"] -Identity $Versions[$i].id -Force
    }
}


try {
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Site"

    $ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
    $collLists = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists -and $_.Title -eq $LibraryName}
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
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing List: $($oList.Title)"

    try{
        $collItems = Get-PnPListItem -List $oList.Title -PageSize 3000
        Add-ScriptLog -Color Cyan -Msg "Collected all Items: $($collItems.Count)"
    }
    catch{
        Add-ScriptLog -Color Red -Msg "Error while processing List '$($oList.Title)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        $ItemCounter++
        continue
    }

    $SubItemCounter = 0
    ForEach($oItem in $collItems) {
        
        $SubItemCounter++
        $PercentCompleteSub = [math]::Round($PercentComplete + ($SubItemCounter * $ItemCounterStep / ( $collItems.Count + 1) * 100), 2)
        Add-ScriptLog -Color Yellow -Msg "$($PercentCompleteSub)% Completed - Processing Item: $($oItem["FileRef"])"

        If($oList.BaseType -eq "DocumentLibrary") {

            If($oItem.FileSystemObjectType -eq "Folder") {Continue}

            $FileVersions = Get-PnPFileVersion -Url $oItem["FileRef"]
            $VersionsToDelete = $FileVersions.Count - $VersionsToKeep

            If($VersionsToDelete -le 0) {
                Add-ReportRecord -List $oList -Item $oItem -VersionsCount $FileVersions.Count -DeletedVersionsCount 0 -VersionDeletedSizeMB 0
            }
            Else {
                $FileSizeTotalMB = [Math]::Round(($oItem["SMTotalSize"].LookupId/1MB),1)
                $VersionDeletedSizeMB = ($FileSizeTotalMB / ( $FileVersions.Count + 1) ) * $VersionsToDelete
                
                For($i=0; $i -lt $VersionsToDelete; $i++)
                {
                    Add-ScriptLog -Color Magenta -Msg "Removing verion: $($FileVersions[$i].VersionLabel)"
                    Remove-PnPFileVersion -Url $oItem["FileRef"] -Identity $FileVersions[$i].id -Force
                }
                Add-ReportRecord -List $oList -Item $oItem -VersionsCount $FileVersions.Count -DeletedVersionsCount $VersionsToDelete -VersionDeletedSizeMB $VersionDeletedSizeMB
            }
        }
        elseIf($oList.BaseType -eq "GenericList") {
            Add-ReportRecord -List $oList -Item $oItem -VersionsCount 0 -DeletedVersionsCount 0 -VersionDeletedSizeMB 0
        }
    }
    $ItemCounter++
}

if($collLists.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collLists.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## Delete versions of all Files in all Document Libraries of a Site

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME"
$VersionsToKeep = 2



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord {
    param (
        $List,
        $Item,
        $VersionsCount,
        $DeletedVersionsCount,
        $VersionDeletedSizeMB
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        ListName = $List.Title
        ListURL = $List.DefaultViewUrl -Replace ('Forms/AllItems.aspx','') -Replace ('AllItems.aspx','')
        ListType = $List.BaseType
        ItemID = $Item.Id
        ItemURL = $Item["FileRef"]
        VersionNo = $Item["_UIVersionString"]
        VersionsHistoryCount = $VersionsCount
        DeletedVersionsCount = $DeletedVersionsCount
        TotalDeletedSizeMB = $VersionDeletedSizeMB
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
$ReportName = "VersionsDeleted"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
Function Remove-Versions {
    param (
        $Item,
        $Versions
    )
 
    For($i=0; $i -lt $VersionsToDelete; $i++)
    {
        Add-ScriptLog -Color Magenta -Msg "Removing verion: $($Versions[$i].VersionLabel)"
        Remove-PnPFileVersion -Url $Item["FileRef"] -Identity $Versions[$i].id -Force
    }
}


try {
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Site"

    $ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
    $collLists = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists}
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
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing List: $($oList.Title)"

    try{
        $collItems = Get-PnPListItem -List $oList.Title -PageSize 3000
        Add-ScriptLog -Color Cyan -Msg "Collected all Items: $($collItems.Count)"
    }
    catch{
        Add-ScriptLog -Color Red -Msg "Error while processing List '$($oList.Title)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        $ItemCounter++
        continue
    }

    $SubItemCounter = 0
    ForEach($oItem in $collItems) {
        
        $SubItemCounter++
        $PercentCompleteSub = [math]::Round($PercentComplete + ($SubItemCounter * $ItemCounterStep / ( $collItems.Count + 1) * 100), 2)
        Add-ScriptLog -Color Yellow -Msg "$($PercentCompleteSub)% Completed - Processing Item: $($oItem["FileRef"])"

        If($oList.BaseType -eq "DocumentLibrary") {

            If($oItem.FileSystemObjectType -eq "Folder") {Continue}

            $FileVersions = Get-PnPFileVersion -Url $oItem["FileRef"]
            $VersionsToDelete = $FileVersions.Count - $VersionsToKeep

            If($VersionsToDelete -le 0) {
                Add-ReportRecord -List $oList -Item $oItem -VersionsCount $FileVersions.Count -DeletedVersionsCount 0 -VersionDeletedSizeMB 0
            }
            Else {
                $FileSizeTotalMB = [Math]::Round(($oItem["SMTotalSize"].LookupId/1MB),1)
                $VersionDeletedSizeMB = ($FileSizeTotalMB / ( $FileVersions.Count + 1) ) * $VersionsToDelete
                
                For($i=0; $i -lt $VersionsToDelete; $i++)
                {
                    Add-ScriptLog -Color Magenta -Msg "Removing verion: $($FileVersions[$i].VersionLabel)"
                    Remove-PnPFileVersion -Url $oItem["FileRef"] -Identity $FileVersions[$i].id -Force
                }
                Add-ReportRecord -List $oList -Item $oItem -VersionsCount $FileVersions.Count -DeletedVersionsCount $VersionsToDelete -VersionDeletedSizeMB $VersionDeletedSizeMB
            }
        }
        elseIf($oList.BaseType -eq "GenericList") {
            Add-ReportRecord -List $oList -Item $oItem -VersionsCount 0 -DeletedVersionsCount 0 -VersionDeletedSizeMB 0
        }
    }
    $ItemCounter++
}

if($collLists.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collLists.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```