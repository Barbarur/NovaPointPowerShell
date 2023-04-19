#Automation #SiteCollection #PowerShell #PnP #CSV 

<br>

## Preparations

1.  Ensure your **PowerShell** is [ready to work with all the necessary modules installed](https://novacato.com/get-powershell-ready-before-start-to-work/).
2.  Check the [error code list](https://novacato.com/tag/error-code/) in case you meet any.

<br>

## Create CSV file with folder structure

| SiteName | SiteURL | NewSiteName | NewSiteUrl |
| :--- | :--- | :--- | :--- |
| SiteNameOne | https://DOMAIN.sharepoint.com/sites/SITEONE | NewSiteNameOne | https://DOMAIN.sharepoint.com/sites/SITEONE |
| SiteNameTwo | https://DOMAIN.sharepoint.com/sites/SITETWO | NewSiteNameTwo | https://DOMAIN.sharepoint.com/sites/SITETWO |
| SiteNameThree | https://DOMAIN.sharepoint.com/sites/SITETHREE | NewSiteNameThree | https://DOMAIN.sharepoint.com/sites/SITETHREE |


<br>

## Run Script

```powershell
# Define Parameters
$AdminSiteURL = "https://DOMAIN-admin.sharepoint.com"
$CSVFile = "$Env:USERPROFILE\Desktop\SiteList.csv"

# Connect to SharePoint Online
Connect-SPOService -Url $AdminSiteURL

# Import User List
$CSVData = Import-CSV $CSVFile

# Loop through each user
$ItemCounter = 0 
ForEach($Row in $CSVData){
    # Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$CSVData.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "User: '$($Row.SiteURL)"

    Start-SPOSiteRename -Identity $Row.SiteURL -NewSiteUrl $Row.NewSiteURL -NewSiteTitle $Row.NewSiteName -Confirm:$false

}
# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Row.URL)"

Write-Host -b Green "Finished renaming sites!"
```