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
    $Global:Results += New-Object PSObject -Property ([ordered]@{
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

## Delete versions of all Files in in all Document Libraries of a Site

```powershell
# Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME"
$VersionsToKeep = 2
$ReportOutput = "$Env:USERPROFILE\Desktop\VersionsDeletedReport.csv"

Connect-PnPOnline -Url $SiteURL -Interactive

Function Add-Report(){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        "List Name"                 = $ListName
        "List URL"                  = $ListRelativeURL
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

$Global:Results = @()

# Get all list and iterate
$ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
$CollectionLists = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists}
ForEach ($List in $CollectionLists)
{
    
    # Report List information
    $ListName               = $List.Title
    $ListRelativeURL        = $List.DefaultViewUrl -Replace ('Forms/AllItems.aspx','') -Replace ('AllItems.aspx','')
    $ListType               = $List.BaseType
    
    $Msg = "Collecting items on {0} '{1}'... " -f $List.BaseType,$List.Title
    Write-Host -F Yellow $Msg

    # Get All Items from the List and iterate
    $ListItems = Get-PnPListItem -List $List.Title -PageSize 3000
    $ItemCounter = 0
    
    ForEach($Item in $ListItems)
    {
        # Report Item infomation
        $ItemID              = $Item.Id
        $ItemRelativeURL     = $Item["FileRef"]
        $Version             = $Item["_UIVersionString"]
        $Versions = Get-PnPProperty -ClientObject $Item -Property Versions
        $VersionsqQty         = $Versions.Count
        
        # Process notification start 
        $ItemCounter++
        $PercentComplete = [math]::Round($ItemCounter/$ListItems.Count*100,1)
        Write-Progress -PercentComplete $PercentComplete -Activity "$Msg $PercentComplete%" -Status $ItemRelativeURL

        # Report Library items unique information
        If($List.BaseType -eq "DocumentLibrary")
        {

            If($Item.FileSystemObjectType -eq "Folder") {Continue}

            $ItemName            = $Item["FileLeafRef"]

            $ListVersions = Get-PnPFileVersion -Url $Item["FileRef"]
            $VersionsToDelete = $ListVersions.Count - $VersionsToKeep
            
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

    }
    
    # Process notification finish
    Write-Progress -Activity "Processing $($Global:ItemCounter)%" -Status $Msg -Completed

}


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