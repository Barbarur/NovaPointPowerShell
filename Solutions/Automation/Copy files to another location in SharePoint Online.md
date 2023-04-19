#Automation #SharePointOnline #PowerShell #PnP #SiteCollection 

<br>

## Option 1; Copy all content inside the source folder location to another location. Using PnP

```powershell
#Define Parameters
$SourceSiteURL = "https://{DOMAIN}.sharepoint.com/sites/{SITENAME}"
$SourceFolderPath = "/Shared Documents/{FOLDERNAME}"
$DestFolderRelatPath = "/sites/{SITENAME}/Shared Documents/{FOLDERNAME}"

#Connect to PnP Online
Connect-PnPOnline -Url $SourceSiteURL -UseWebLogin

#Collect Items to copy
$ItemsList = Get-PnPFolderItem -FolderSiteRelativeUrl $SourceFolderPath

$ItemCounter = 0 
ForEach($Item in $ItemsList)
{
    Copy-PnPFile -SourceUrl $Item.ServerRelativeUrl -TargetUrl $DestFolderRelatPath -Force -OverwriteIfAlreadyExists
            
    $ItemCounter++
    Write-Host -f Yellow "Copying file: "$Item.Name
    Write-Progress -PercentComplete ($ItemCounter / ($ItemsList.Count) * 100) -Activity "Processing Files $ItemCounter of $($ItemsList.Count)" -Status "Getting data from File '$($Item.Name)"
}
```

<br>

## Option 2; Copy the full source folder to another location. Using PnP

```powershell
#Define Parameters
$SiteURL = "https://{DOMAIN}-my.sharepoint.com/personal/alexw_grizzled_onmicrosoft_com/"
$SourceFolderURL = "/personal/alexw_grizzled_onmicrosoft_com/Documents/AA"
$TargetFolderURL = "/personal/alland_grizzled_onmicrosoft_com/Documents/Import"
  
#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin
  
#Copy All Files and Folders between source and target folders
Copy-PnPFile -SourceUrl $SourceFolderURL -TargetUrl $TargetFolderURL -Force -OverwriteIfAlreadyExists
```