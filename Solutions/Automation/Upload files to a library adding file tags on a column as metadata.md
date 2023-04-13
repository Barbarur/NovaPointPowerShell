#Automation #PnP #PowerShell #SiteCollection #TagColumn #DocumentLibrary

<br>

```powershell
#Config Variables
$SiteURL = "https://DOMAIN.sharepoint.com/sites/SITENAME"
$DestinationPath = "LIBRARYNAME"
$ColumnName = "ColumnTitle"
$SourceFilePath ="PictureFolderPath" # i.e. C:\Users\USERNAME\OneDrive\PICTUREFOLDER\
  
#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin

# Set item processing counter to 0
$ItemCounter = 0

#Collect all Items and iterate
$ListItems = Get-ChildItem -Path $SourceFilePath
ForEach($Item in $ListItems){
    # Progress notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$ListItems.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Item '$($item.FullName)"

    # Extract Tag from File
    $folder = (New-Object -ComObject Shell.Application).NameSpace((Split-Path $item.FullName))
    $Tag = $folder.GetDetailsOf($folder.ParseName((Split-Path -Leaf $item.FullName)),18)

    # Upload file to a the document library
    Add-PnPFile -Path $item.FullName -Folder $DestinationPath -Values @{$ColumnName = $Tag}

}
#Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($item.FullName)"
```