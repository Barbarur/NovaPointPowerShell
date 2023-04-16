#Report #SharePointOnline #PowerShell #PnP #DocumentLibrary #ItemList #FileSize #Versioning 

<br>

```powershell
# Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME"
$ReportOutput = "C:\Temp\SiteFilesReport.csv"

$Global:Results = @()

# Add new log to report function
Function Add-Report(){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        "List Name"            = $ListName
        "List URL"             = $ListRelativeURL
        "Item ID"              = $ItemID
        "Item Name"            = $ItemName
        "Item URL"             = $ItemRelativeURL
        "Item Type"            = $ItemType
        Created                = $Created
        "Created by"           = $CreatedBy
        Modified               = $Modified
        "Modified by"          = $ModifiedBy
        "Version No"           = $Version
        "Versions Qty"         = $VersionsqQty
        "File Size (MB)"       = $FileSizeMB
        "Total File Size (MB)" = $FileSizeTotalMB
        })
    }

# Get Credentials
$Cred = Get-Credential

# Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -Credentials $Cred

# Get all list and iterate
$ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
$Lists = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists}
ForEach ($List in $Lists)
    {
    # Process notification parameters
    $ItemCounter = 0
    $Msg = "Collecting items on {0} '{1}'... " -f $List.BaseType,$List.Title
    Write-Host -f Yellow $Msg -NoNewline
    
    # Report List information
    $ListName               = $List.Title
    $ListRelativeURL        = $List.DefaultViewUrl -Replace ('Forms/AllItems.aspx','') -Replace ('AllItems.aspx','')
    $ListType               = $List.BaseType
    
    $Items = Get-PnPListItem -List $List.Title -PageSize 2000
    
    ForEach($Item in $Items)
        {
        # Report Item infomation
        $ItemID              = $Item.Id
        $ItemRelativeURL     = $Item["FileRef"] -Replace ($ListRelativeURL,'')
        $Created             = $Item["Created"]
        $CreatedBy           = $Item["Author"].Email
        $Modified            = $Item["Modified"]
        $ModifiedBy          = $Item["Editor"].Email
        $Version             = $Item["_UIVersionString"]
        $Versions = Get-PnPProperty -ClientObject $Item -Property Versions
        $VersionsqQty         = $Versions.Count
        
        # Process notification start 
        $ItemCounter++
        $ItemProcess = [math]::Round($ItemCounter/$Items.Count*100,1)
        Write-Progress -PercentComplete $ItemProcess -Activity "$Msg $ItemProcess%" -Status $ItemRelativeURL

        # Report Library items unique information
        If($List.BaseType -eq "DocumentLibrary")
            {
            If($Item.FileSystemObjectType -eq "Folder") {Continue}

            $ItemName            = $Item["FileLeafRef"]
            $FileSizeMB          = [Math]::Round(($Item["File_x0020_Size"]/1MB),1)
            $FileSizeTotalMB     = [Math]::Round(($Item["SMTotalSize"].LookupId/1MB),1)

            Add-Report
            }
        # Report List items unique information
        If($List.BaseType -eq "GenericList")
            {
            $ItemName            = $Item["Title"]
            $Attachments = Get-PnPProperty -ClientObject $Item -Property AttachmentFiles
            $AttachmentFileSizeTotal = 0
            ForEach($Attachment in $Attachments)
                {
                $AttachmentFile = Get-PnPFile -Url $Attachment.ServerRelativeUrl -AsFileObject
                $AttachmentFileSizeTotal += $AttachmentFile.Length
                }
            $FileSizeMB           = [Math]::Round(($AttachmentFileSizeTotal/1MB),1)
            $FileSizeTotalMB      = $FileSizeMB

            Add-Report
            }
        }
    
    # Process notification finish
    Write-Host -f Green "COMPLETED!"
    Write-Progress -Activity "Processing $($Global:ItemCounter)%" -Status $Msg -Completed
    }


# Export the results to CSV
If($Global:Results.count -eq 0){
    Write-host -b Red "Report is empty!"
}
Else{
    If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
    $Global:Results | Export-Csv -Path $ReportOutput -NoTypeInformation
    Write-host -b Green "Report Generated Successfully!"
    Write-host -f Green $ReportOutput
}
```