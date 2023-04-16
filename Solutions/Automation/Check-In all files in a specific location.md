#Automation #PowerShell #PnP #DocumentLibrary #SiteCollection #Subsite 

<br>

```powershell
#Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME/"
$ListName = "Documents"
$FolderServerRelativeURL = "/sites/SITENAME/Shared Documents/FOLDERNAME"
 
#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -Credentials (Get-Credential)
 
#Get All List Items from the List - Filter Files
$ListItems = Get-PnPListItem -List $ListName -FolderServerRelativeUrl $FolderServerRelativeURL -PageSize 500 | Where {$_["FileLeafRef"] -like "*.*"}
 
#Loop through each list item
ForEach ($Item in $ListItems)
{
    Write-host -f Yellow "Testing If file is Checked-Out:"$Item.FieldValues["FileRef"]
    #Get the File from List Item
    $File = Get-PnPProperty -ClientObject $Item -Property File
 
    If($File.Level -eq "Checkout")
    {
        #Check-In and Approve the File
        Set-PnPFileCheckedIn -Url $File.ServerRelativeUrl -CheckinType MajorCheckIn
 
        Write-host -f Green "`tFile Checked-In:"$File.ServerRelativeUrl
    }
}
```