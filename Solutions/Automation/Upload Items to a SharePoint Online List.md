#Automation #PowerShell #PnP #ItemList 

<br>

```powershell
#Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME"
$ListName = "LISTNAME"
$CSVFile = "C:\Temp\ItemsToUpload.csv"

$CSVData = Import-CSV $CSVFile

#Connect to PNP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin

ForEach($Row in $CSVData)
    {
    Add-PnPListItem -List $ListName -Values @{"Title" = $Row.'Title' ; "Column#1"=$Row.'Column1'; "Column#2"=$Row.'Column2'; "Column#3"=$Row.'Column3'; "Column#4"=$Row.'Column4'}
    }
Write-Host -f Green "Done"
```