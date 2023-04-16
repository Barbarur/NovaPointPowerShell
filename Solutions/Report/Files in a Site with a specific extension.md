#Report #SharePointOnline #OneDrive #PowerShell #PnP #DocumentLibrary

<br>

```powershell
# Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME"
$FileExt = "DOCX"
$ReportOutput = "$Env:USERPROFILE\Desktop\FileExtReport.csv"

$Global:Results = @()

# Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -Interactive

# Get all list and iterate
$ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
$Lists = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists -and $_.BaseType -eq "DocumentLibrary"}
ForEach ($List in $Lists)
    {

    # Process notification parameters
    $ItemCounter = 0
    $Msg = "Collecting items on {0} '{1}'... " -f $List.BaseType,$List.Title
    Write-Host -f Yellow $Msg -NoNewline
    
    # Report List information
    $ListName               = $List.Title
    $ListRelativeURL        = $List.DefaultViewUrl -Replace ('Forms/AllItems.aspx','') -Replace ('AllItems.aspx','')
    
    $ListItems = Get-PnPListItem -List $List.Title -PageSize 2000 | Where-Object { $_["FileLeafRef"] -like "*.$FileExt" }
    
    ForEach($Item in $ListItems)
    {
        
        $ItemRelativeURL     = $Item["FileRef"] -Replace ($ListRelativeURL,'')
        
        # Process notification start 
        $ItemCounter++
        $PercentComplete = [math]::Round($ItemCounter/$ListItems.Count*100,1)
        Write-Progress -PercentComplete $PercentComplete -Activity "$Msg $PercentComplete%" -Status $ItemRelativeURL


        If($Item.FileSystemObjectType -eq "Folder") {Continue}

        $Versions = Get-PnPProperty -ClientObject $Item -Property Versions
        $VersionsqQty         = $Versions.Count
        $FileSizeMB          = [Math]::Round(($Item["File_x0020_Size"]/1MB),1)

        $Global:Results += New-Object PSObject -Property ([ordered]@{
        "List Name"            = $ListName
        "List URL"             = $ListRelativeURL
        "Item ID"              = $Item.Id
        "Item Name"            = $Item["FileLeafRef"]
        "Item URL"             = $ItemRelativeURL
        Created                = $Item["Created"]
        "Created by"           = $Item["Author"].Email
        Modified               = $Item["Modified"]
        "Modified by"          = $Item["Editor"].Email
        "Version No"           = $Item["_UIVersionString"]
        "Versions Qty"         = $VersionsqQty
        "File Size (MB)"       = $FileSizeMB
        })

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